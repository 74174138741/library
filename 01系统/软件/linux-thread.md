# Linux 线程与同步

进程像一个部门：有自己的办公室（堆、全局变量、文件描述符）。线程是部门里的人：共享办公室，但每人有自己的草稿本（栈）和现场（寄存器）。

所以线程之间「通信」几乎总是 **共享内存 + 同步**。不用管道也能说话，但一不锁就会抢同一支笔。

和进程篇对照：跨进程要 IPC；同进程多线程，默认已经在同一块内存里。

---

## 1. 并发不是自动变快

- 单核上多线程是**交替**（并发），不是同时算。
- 多核上才可能**真并行**。超线程是一个物理核装两个逻辑核，吞吐有帮助，不如真多核。
- Python `threading` 有 GIL：CPU 密集算不了多核；等网络、等磁盘时仍然有用。
- C++ 用标准库 `<thread>`（C++11 起）。

```cpp
std::thread t(worker, 1, 5);   // 函数 + 参数
t.join();                      // 等它结束；可 join 的线程必须 join 或 detach 之一
t.detach();                    // 放飞，之后不能再 join
std::this_thread::sleep_for(std::chrono::milliseconds(100));
std::this_thread::yield();     // 暗示调度器：我可以让一下
```

主函数结束进程就没了，`detach` 出去的线程也会被干掉。要靠谱收场，优先 `join`。

---

## 2. 先认清三种事故

| 现象 | 是什么 |
|---|---|
| 竞争 (race) | 两人同时改一个变量，结果看调度运气 |
| 死锁 | A 拿锁1等锁2，B 拿锁2等锁1 |
| 过度争用 | 锁太热，线程都在排队，比单线程还慢 |

对策对应三种工具：互斥锁、条件变量（按条件等）、读写锁（读可以一起，写独占）。计数、标志位能用原子操作就别上大锁。要限制「同时进来几个人」用信号量。

---

## 3. 锁

`std::mutex`：同一时刻只有一个线程能拿着。

```cpp
std::mutex mtx;
int counter = 0;

void worker() {
    for (int i = 0; i < 1000; i++) {
        std::lock_guard<std::mutex> lock(mtx);  // 构造加锁，析构解锁
        counter++;
    }
}
```

不加锁时 `counter` 最后不一定是 2000，每次还可能不一样。

两种守卫：

| | `lock_guard` | `unique_lock` |
|---|---|---|
| 加锁 | 构造立刻 | 可以推迟、可以手动 |
| 解锁 | 只能等析构 | 可以中途 `unlock` 再 `lock` |
| 用途 | 一段临界区 | 条件变量必须用它 |

读多写少用 `std::shared_mutex`：`lock_shared()` 多人读，`lock()` 独占写。

多个锁时**所有线程按同一顺序加锁**，避免死锁。

---

## 4. 条件变量：线程之间的 if

锁保证「同一时刻只有一人改队列」。条件变量保证「空了就睡，来货再醒」，避免空转。

```cpp
std::mutex mtx;
std::condition_variable cv;
std::queue<int> q;

void producer() {
    for (int i = 1; i <= 5; i++) {
        std::unique_lock<std::mutex> lock(mtx);
        q.push(i);
        cv.notify_one();          // 叫醒一个在 wait 的人；没人等就丢了
    }
}

void consumer() {
    for (int i = 0; i < 5; i++) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, []{ return !q.empty(); });  // 谓词为假则解锁并睡
        int val = q.front(); q.pop();
    }
}
```

`wait` 被唤醒后会先重新加锁，再检查谓词。谓词仍假就继续睡（防止假醒）。必须用 `unique_lock`，因为 wait 内部要能解锁。

---

## 5. 原子操作

`i++` 在 CPU 上是「读出 → 加一 → 写回」三步，中间能被插队。`std::atomic` 把这三步收成一步。适合计数器和开关，不适合保护一整段业务逻辑。

```cpp
std::atomic<int> counter{0};
counter.fetch_add(1);
```

---

## 6. 信号量：停车场

计数器 = 空车位。`acquire` 进来占一个（到 0 就在门口等），`release` 开走还一个。用来**限制并发数量**，不是用来替代互斥语义的复杂临界区。

经典：有界队列里两个信号量 + 一把锁。

```
empty 初值 = 容量      还有几个空位
full  初值 = 0         已有几份数据
mutex                  保护队列结构本身
```

```cpp
std::counting_semaphore<5> empty(5);
std::counting_semaphore<5> full(0);
std::mutex mtx;
std::queue<int> buffer;

void producer() {
    empty.acquire();
    { std::lock_guard<std::mutex> lock(mtx); buffer.push(x); }
    full.release();
}
void consumer() {
    full.acquire();
    { std::lock_guard<std::mutex> lock(mtx); buffer.pop(); }
    empty.release();
}
```

（`std::counting_semaphore` 是 C++20。）共享内存那一侧跨进程同步，用的是 POSIX/System V 信号量，意思一样：计数 + 等。

---

## 7. 线程池

不要来一个任务就 `std::thread` 一次。池子里养固定几个人，外面只往队列丢活。

```
提交 enqueue ──► 任务队列 ──► 空闲 worker 取出执行
                     ▲
                     └── 队列空则 wait；析构时 stop=true，notify_all，join
```

核心就三样：worker 数组、任务队列、一把锁 + 条件变量。精简骨架：

```cpp
class ThreadPool {
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queueMutex;
    std::condition_variable condition;
    bool stop = false;
public:
    explicit ThreadPool(size_t n) {
        for (size_t i = 0; i < n; ++i) {
            workers.emplace_back([this] {
                for (;;) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(queueMutex);
                        condition.wait(lock, [this] { return stop || !tasks.empty(); });
                        if (stop && tasks.empty()) return;
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();
                }
            });
        }
    }
    void enqueue(std::function<void()> task) {
        {
            std::lock_guard<std::mutex> lock(queueMutex);
            tasks.emplace(std::move(task));
        }
        condition.notify_one();
    }
    ~ThreadPool() {
        { std::lock_guard<std::mutex> lock(queueMutex); stop = true; }
        condition.notify_all();
        for (auto& w : workers) if (w.joinable()) w.join();
    }
};
```

工业线程池还会加：队列上限、拒绝策略、非核心线程超时回收。先把「队列 + wait/notify + join」做对，那些是加料。
