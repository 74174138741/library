# Linux 进程与 IPC

程序是磁盘上的一堆字节。进程是它跑起来之后，内核配给的一份**独立资源**：地址空间、文件描述符、网络连接。Linux 上**资源以进程为单位分配，CPU 以线程为单位调度**。单线程进程里，两者碰巧是同一个东西。

进程之间内存默认隔离。要说话，必须走内核提供的通道——这就是 IPC。ROS 的 Topic/Service/Action 是更上一层的约定，底下仍然是这类通道（再往下常常是 DDS/套接字/共享内存）。

---

## 1. 进程长什么样

```
父进程
  ├─ fork() 复制出子进程（写时复制：没改的页还共享物理内存）
  │     └─ exec() 把子进程的代码换成新程序（PID 不变）
  └─ wait() 收回子进程的退出状态，否则它变成僵尸
```

- **孤儿**：父进程先死，子进程被 `init`/`systemd` 收养。
- **僵尸**：子进程已退出，父进程还没 `wait`，内核里留着一行「尸位」记账。
- **守护进程**：不挂终端、登出了还在，通常由 systemd 管。`ps` 里 `TTY` 为 `?` 的多半是它们。`sshd`、`cron` 都是。

状态（`ps` 里那一列）：

| 字母 | 含义 |
|---|---|
| `R` | 正在跑或排队要跑 |
| `S` | 可中断睡眠，等事件（常见） |
| `D` | 不可中断睡眠，多半等磁盘；这时 `kill -9` 也得等 I/O 完 |
| `T` | 被暂停（调试 / `SIGSTOP`） |
| `Z` | 僵尸 |

```bash
ps -ef                  # 全进程，完整命令行；越靠下通常越新
ps -ef --forest         # 树：谁叉出了谁
kill -15 4000           # SIGTERM，请它自己收拾后退出（优先）
sudo kill -9 4000       # SIGKILL，不能捕获，当场死
sleep 10                # 当前 shell 睡 10 秒
```

`PID` 自己，`PPID` 父亲，`UID` 谁的，`CMD` 命令行。

---

## 2. 生与死：fork / exec / wait / exit

`fork()` 返回三次语义：父进程里是子 PID，子进程里是 `0`，失败是 `-1`。

`exec` 系列**不创建新进程**，只把当前进程的代码换成另一个程序。成功则后面的语句永远不会跑；失败才继续，所以失败分支里要处理错误。

名字规则：

- `l` 参数列表 / `v` 参数数组
- `p` 从 `PATH` 里找
- `e` 自己传环境变量

典型组合：父进程 `fork`，子进程 `exec`，父进程 `wait`——这样父进程自己的逻辑还在。

```c
pid_t pid = fork();
if (pid == 0) {
    execl("/bin/ls", "ls", NULL);
    exit(1);              // 只有 exec 失败才到这里
} else {
    wait(NULL);
    printf("父进程继续\n");
}
```

回收：

```c
waitpid(pid, &status, 0);          // pid>0 等指定；-1 任意子进程
waitpid(-1, &status, WNOHANG);     // 不阻塞：没人退出就返回 0
```

`WUNTRACED` / `WCONTINUED` 还会在子进程被暂停、被继续时返回。进程自己收工：`exit(0)`。

---

## 3. 先选 IPC，再写代码

和 ROS 选频道一样：先问「有没有亲缘、要不要拷贝、要不要同步」。

| | 管道 pipe | 命名管道 FIFO | 消息队列 | 共享内存 | 套接字 | 信号 |
|---|---|---|---|---|---|---|
| 谁能用 | 有亲缘（父子/兄弟） | 任意进程，靠路径 | 任意，靠 key | 任意，靠名字/key | 本机或跨机 | 任意，靠 PID |
| 像什么 | 一根匿名水管 | 水管做成了文件 | 带类型的信箱 | 同一块黑板 | 打电话 | 拍一下肩膀 |
| 拷贝 | 进内核再出来 | 同左 | 同左 | 几乎零拷贝 | 进内核 | 几乎不带数据 |
| 同步 | 内核管读写阻塞 | 没人读/写会阻塞 | 可阻塞 | **不管**，要自己加锁 | 连接+读写 | 异步打断 |

口诀：

- 父子临时递一句 → **pipe**
- 两个无关程序、当文件用 → **FIFO**
- 要按类型取消息 → **消息队列**
- 控制环、硬件状态、嫌拷贝慢 → **共享内存 + 锁/信号量**
- 以后可能跨机器，或要双向流 → **套接字**
- 只想说「停 / 继续 / 自定义一拍」→ **信号**

---

## 4. 管道 pipe

内核里一块缓冲区，`fd[0]` 读、`fd[1]` 写，单向。`fork` 之后两边都拿到同一对 fd，各关不用的那一端。

```c
int fd[2];
pipe(fd);
pid_t pid = fork();
if (pid > 0) { close(fd[0]); write(fd[1], msg, len); close(fd[1]); wait(NULL); }
else         { close(fd[1]); read(fd[0], buf, sizeof buf); close(fd[0]); }
```

shell 的 `|` 就是它：左边进程的 stdout 接到管道写端，右边的 stdin 接到读端。

---

## 5. FIFO（命名管道）

和普通文件一样 `open` / `read` / `write`，但内容是管道而不是磁盘。路径是约定，无关进程靠这个见面。

没写端时读会阻塞，没读端时写会阻塞。字节流，先进先出。

```c
mkfifo("/tmp/myfifo", 0666);
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "Hello from FIFO!", 17);
close(fd);
```

另一进程 `open(..., O_RDONLY)` 再 `read`。

---

## 6. 共享内存

最快：双方映射同一块物理页，写完对面立刻能看见。内核不当邮差，所以**自己必须配锁**，否则就是数据竞争。

两条 API，别混：

**POSIX**（名字是字符串，像文件）：

```c
int fd = shm_open("/my_posix_shm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
char *p = mmap(0, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// 用完
munmap(p, 4096);
shm_unlink("/my_posix_shm");
```

`mmap` 参数：地址让系统选、长度、读写权限、`MAP_SHARED` 表示改动别人看得到、从对象开头映射。

**System V**（工业代码里常见，用整数 key）。机器人里经常拿一块结构体当硬件接口：

```c
shmid = shmget((key_t)10000, sizeof(SHM_HARDWARE_INTERFACE), 0666 | IPC_CREAT);
shm = (SHM_HARDWARE_INTERFACE *)shmat(shmid, NULL, 0);
shm->data = 42;
shmdt(shm);          // 只是本进程松开，块还在
// 真正删：shmctl(shmid, IPC_RMID, NULL);
```

读的一方 `shmget` **不要**带 `IPC_CREAT`。同步用信号量或把 `pthread_mutex` 放进这块内存（要设 `PTHREAD_PROCESS_SHARED`）。

---

## 7. 消息队列

按「类型」取信，不必按到达顺序。结构体第一个字段必须是 `long msg_type`，且 `> 0`。

```c
msgid = msgget(0x1234, IPC_CREAT | 0666);
msgsnd(msgid, &msg, sizeof msg.msg_text, 0);           // 0 = 阻塞
msgrcv(msgid, &msg, sizeof msg.msg_text, 1, 0);        // 只要 type==1
msgctl(msgid, IPC_RMID, NULL);
```

`msgrcv` 第四个参数：`>0` 指定类型；`0` 队列里第一条；`<0` 类型 ≤ 绝对值的最小那种。最后一参 `IPC_NOWAIT` 为非阻塞。

---

## 8. 套接字

本机两个进程也能用，换 `AF_INET` 就能上网络。服务端四步：`socket` → `bind` → `listen` → `accept`，然后 `read`/`write`。客户端把 `bind/listen/accept` 换成 `connect`。

```c
server_fd = socket(AF_INET, SOCK_STREAM, 0);
bind(server_fd, ...);
listen(server_fd, 5);
client_fd = accept(server_fd, ...);
read(client_fd, buf, sizeof buf);
write(client_fd, "已收到\n", ...);
```

同机更轻的是 `AF_UNIX` 路径套接字，语义一样。

---

## 9. 信号

很瘦的异步通知，几乎不带载荷。适合「停、继续、自定义拍一下」，不适合传结构体。

| 类 | 例子 | 备注 |
|---|---|---|
| 终止 | `SIGTERM`(15) 礼貌关；`SIGKILL`(9) 强制 | KILL 不能捕获 |
| 错误 | `SIGSEGV` 非法内存；`SIGFPE` 除零 | 内核因你犯错而发 |
| 控制 | `SIGSTOP` 暂停；`SIGCONT` 继续 | STOP 也不能捕获 |
| 自定义 | `SIGUSR1` / `SIGUSR2` | 进程间约定含义 |
| 实时 | `SIGRTMIN`…`SIGRTMAX` | 可排队、可带一点数据 |

进程对信号：默认动作、忽略、或自己注册处理函数。`kill` 命令就是在发信号。
