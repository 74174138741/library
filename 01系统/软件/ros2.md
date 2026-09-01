# ROS 2

ROS 2 不是操作系统。它是一套**进程间通信框架**：每个功能做成独立节点（进程），彼此不共享内存、不直接调函数，只通过三种频道交换数据。

非硬实时的模块间通信，ROS 2 就是现在的普通话，频率大概在几 Hz 到一两百 Hz，偶尔丢一帧、延迟抖几毫秒可以接受

底层是 DDS（去中心化，没有 ROS 1 那种 Master）。同一台机器、局域网里的节点会自动发现对方。真正要掌握的只有一件事：**选对频道，把信封格式说清楚。**

官方：[docs.ros.org/en/jazzy](https://docs.ros.org/en/jazzy/) · 源码组织：[github.com/ros2](https://github.com/ros2)

---

## 1. 心智模型：一群不会串门的小程序

把机器人想成几个各干各的程序：

```
雷达节点 ──┐
键盘节点 ──┼──  ROS 通信总线  ──►  底盘节点
规划节点 ──┘                    ──►  可视化
```

它们之间只认识**频道名**和**消息类型**，不认识对方是谁、在哪台机器上。换掉雷达、加一个录包节点，底盘代码不用改。

三种频道，对应三种说话方式：

| 频道 | 像什么 | 方向 | 等不等回复 | 典型用途 |
|---|---|---|---|---|
| **Topic** | 电台广播 | 单向，可多人听 | 不等，发完就走 | 传感器、速度、图像，一直在流 |
| **Service** | 打电话 | 问一句答一句 | 等结果 | 开关、标定、查状态、算一次 |
| **Action** | 派活 + 进度条 | 目标 / 反馈 / 结果 三路 | 边做边报，可取消 | 导航、抓取、轨迹执行 |

信封里装什么，由 **msg / srv / action** 文件定义。语言无关：Python 发、C++ 收，只要类型名一样。

节点本身还有定时器和参数，那是节点的内部工具，不是节点之间的通信。通信只有上面三条。

---

## 2. 先选频道

动手前先问：对方需不需要回话？这件事会不会跑很久？

```
数据是持续喷的？（激光、IMU、cmd_vel、图像）
        │
        ├─ 是 ──────────────────────────────► Topic
        │
        └─ 否，是一次请求
                │
                ├─ 很快结束，只要最终答案 ──► Service
                │     （加个开关、算个 IK、保存地图）
                │
                └─ 要跑一段时间，还想看进度、能取消
                                              ► Action
                    （去某个点、抓东西、整段轨迹）
```

口诀：

- 传感器、控制指令 → **Topic**
- 「帮我做一下，做完告诉我」→ **Service**
- 「去做这件事，做到哪了？算了取消」→ **Action**

不要用 Topic 当开关（发了没人保证处理）；不要用 Service 传 30Hz 的激光（会堵住）。

---

## 3. 节点：通信的插头

节点 = 一个能插上总线的进程。生命周期就四步：

```python
rclpy.init()
node = Node("名字")     # 向总线报到
rclpy.spin(node)        # 卡住听：订阅、服务、动作回调都在这里被叫醒
rclpy.shutdown()
```

`spin` 不是「让程序忙起来」，是**不挂断电话**：没有它，发布者能发，但订阅/服务/动作的回调不会跑。

现场看谁在线上：

```bash
ros2 node list
ros2 node info /某个节点    # 它发哪些话题、提供哪些服务
```

---

## 4. 信封：msg / srv / action

跨进程传的不是 Python 对象，是一份**双方都认的二进制协议**。

```
.msg / .srv / .action 定义
        ↓ 编译生成各语言的类
创建对象、填字段
        ↓ 序列化（CDR）
网络 / 共享内存
        ↓ 反序列化
对面拿到同样结构的对象
```

### 现成类型，优先用

`std_msgs` 是最小积木：`String`、`Float64`、`Int32`、`Bool`、`Header`。机器人里更常见的是：

| 类型 | 典型话题 | 装什么 |
|---|---|---|
| `sensor_msgs/LaserScan` | `/scan` | 一圈激光 |
| `sensor_msgs/Imu` | `/imu` | 加速度 + 角速度 |
| `geometry_msgs/Twist` | `/cmd_vel` | 线速度 + 角速度 |
| `nav_msgs/Odometry` | `/odom` | 位姿估计 |
| `sensor_msgs/Image` | `/image_raw` | 图像 |

`Header` = 时间戳 + `frame_id`，是时间和坐标系的交汇点。带空间含义的数据几乎都应该有它。

字段类型速查：`bool` / `int32` / `float64` / `string` / `float64[]` / `float64[3]`。时间用 `builtin_interfaces/Time`（`sec` + `nanosec`）。

### 自定义消息（Topic 用）

文件：`my_robot_msgs/msg/RobotState.msg`

```
std_msgs/Header header
float64[] joint_positions
string mode
bool is_grasping
```

CMake：

```cmake
find_package(rosidl_default_generators REQUIRED)
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/RobotState.msg"
  DEPENDENCIES std_msgs
)
```

`package.xml` 里要有 `rosidl_default_generators`、`rosidl_default_runtime`、`std_msgs`。用的时候：

```python
from my_robot_msgs.msg import RobotState
msg = RobotState()
msg.joint_positions = [0.1, 0.2]
```

### 自定义服务（一问一答）

文件：`AddTwoInts.srv`，`---` 上面是请求，下面是响应。

```
int64 a
int64 b
---
int64 sum
```

### 自定义动作（目标 / 结果 / 反馈）

文件：`Navigate.action`，两道 `---` 分成三段。

```
geometry_msgs/PoseStamped target
---
bool success
float32 distance_traveled
---
float32 progress
string status
```

---

## 5. Topic：电台

发布者对着频道名喊，订阅者调到同一频道就能听。双方**不必同时在线、不必互相认识**。同一话题可以多个发布者、多个订阅者。

```
键盘 ──发布──► /cmd_vel ──订阅──► 真底盘
                    │
                    └──订阅──► 仿真底盘
                    └──订阅──► 录包
```

### 代码里只记三件事

创建发布者：类型、话题名、队列长度。

```python
self.pub = self.create_publisher(Twist, "cmd_vel", 10)
msg = Twist()
msg.linear.x = 0.5
self.pub.publish(msg)
```

创建订阅者：类型、话题名、收到时调用的函数。

```python
self.sub = self.create_subscription(Twist, "cmd_vel", self.on_cmd, 10)

def on_cmd(self, msg: Twist):
    # 这里才真正「听到」
    v, w = msg.linear.x, msg.angular.z
```

周期往外喷，用节点的定时器（非阻塞，别写 `while` 死循环）：

```python
self.create_timer(0.05, self.tick)   # 20 Hz，Python 单位是秒
```

队列长度 `10`：来不及处理时最多积 10 条，旧的挤掉。图像、激光这种大流量，这个数字和 QoS 比代码逻辑更常出问题。

### 不用写节点也能说话

这是 Topic 最好用的地方：先用命令验证频道，再写代码。

```bash
ros2 topic list                          # 现在有哪些电台
ros2 topic echo /cmd_vel                 # 听这个频道
ros2 topic info /cmd_vel                 # 谁在发、谁在听、什么类型
ros2 topic hz /scan                      # 实际频率
ros2 topic type /cmd_vel                 # 消息类型名

# 自己当发布者（-r 1 表示每秒一次）
ros2 topic pub -r 1 /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.2}, angular: {z: 0.0}}"
```

底盘不动时：先 `echo` 看有没有指令，再 `info` 看有没有人订 `/cmd_vel`。频道通了再查节点内部。

---

## 6. Service：打电话

客户端拨号，服务端必须接、必须回话。同一服务名：**服务端只能有一个，客户端可以有多个。**

```
客户端                         服务端
   │                              │
   │──────── 请求 (a, b) ────────►│
   │                              │  处理…
   │◄─────── 响应 (sum) ──────────│
```

适合：切换模式、保存地图、手眼标定、求一次逆解。不适合：控制环、传感器流。

### 代码要点

服务端：注册名字 + 处理函数，**必须把 response 填好并 return**。

```python
self.create_service(AddTwoInts, "add_two_ints", self.on_add)

def on_add(self, request, response):
    response.sum = request.a + request.b
    return response
```

客户端：先等到服务出现，再异步拨出，在 `spin` 里等 future。

```python
self.cli = self.create_client(AddTwoInts, "add_two_ints")
self.cli.wait_for_service()

req = AddTwoInts.Request()
req.a, req.b = 3, 5
future = self.cli.call_async(req)
rclpy.spin_until_future_complete(self, future)
print(future.result().sum)
```

`wait_for_service` 很关键：服务端还没起来就 call，会一直空等或失败。这不是 Topic「先发后听也行」。

### 命令行当客户端

```bash
ros2 service list
ros2 service type /add_two_ints
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 10, b: 20}"
```

---

## 7. Action：派活

一次任务拆成三条通道，所以它看起来像「Service + 一串 Topic」。

```
客户端                              服务端
  │                                    │
  │──── 目标 Goal（去哪 / 抓谁）──────►│
  │◄─── 接受或拒绝 ────────────────────│
  │                                    │
  │◄═══ 反馈 Feedback（循环，做到哪）══│   执行中
  │                                    │
  │◄─── 结果 Result（成 / 败）─────────│
  │──── 也可以中途 Cancel ────────────►│
```

| 段 | 像什么 |
|---|---|
| Goal | 任务单 |
| Feedback | 进度条，执行期间可多次 |
| Result | 结案报告，只有一次 |
| Cancel | 客户挂电话 |

### 代码要点

服务端在 `execute_callback` 里干活：循环中 `publish_feedback`，随时查 `is_cancel_requested`，**全部做完再** `succeed()` 并 return Result。不要一开始就标成功。

```python
def execute_callback(self, goal_handle):
    name = goal_handle.request.object_name
    feedback = GrabObject.Feedback()

    for i in range(1, 11):
        if goal_handle.is_cancel_requested:
            goal_handle.canceled()
            result = GrabObject.Result()
            result.success = False
            return result
        feedback.progress = i * 10.0
        goal_handle.publish_feedback(feedback)
        time.sleep(0.5)

    goal_handle.succeed()
    result = GrabObject.Result()
    result.success = True
    return result
```

客户端：`send_goal_async(..., feedback_callback=...)`，先等「接不接单」，再等结果；要停就 `cancel_goal_async`。

### 命令行派活

```bash
ros2 action list
ros2 action info /grab_object
ros2 action send_goal /grab_object action_demo/action/GrabObject \
  "{object_name: 'apple'}" --feedback
```

---

## 8. 现场：先看线，再看代码

通信出问题，80% 不是算法，是**名字、类型、域、有没有 spin**。按这个顺序查。

```bash
echo $ROS_DOMAIN_ID          # 两边必须一样，否则互相隐形
ros2 node list               # 节点起来了没
ros2 topic list              # 频道在不在（注意有没有 / 前缀）
ros2 topic info /话题名      # 类型是否一致、有没有 pub/sub
ros2 topic echo /话题名      # 线上到底有没有数据
ros2 interface show geometry_msgs/msg/Twist   # 字段长什么样
```

两台机器听不见对方：先对 `ROS_DOMAIN_ID`，再查防火墙 / 是否同一网段。同一台机器「list 得到、echo 是空」：发布者没 `spin` 或没按频率 `publish`。

画出现在谁连着谁：

```bash
ros2 run rqt_graph rqt_graph
```

---

## 9. 附录：把节点挂上总线

通信是主体。下面只解决「代码怎么变成能 `ros2 run` 的节点」。新终端默认看不见你编译的包，所以每次都要 `source`。

工作空间是一次编译单位：

```
ws/
├── src/        源码（功能包）
├── build/      中间文件，不要手改
├── install/    真正能跑的东西；setup.bash 在这里
└── log/
```

```bash
colcon build
source install/setup.bash          # 改当前终端的 PATH / PYTHONPATH / AMENT_PREFIX_PATH
ros2 run 包名 节点名
```

`source` = 把本工作空间抄进**当前这个终端**的环境变量。新开终端要再 source 一次。自动挂上：

```bash
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
```

改了 C++ / 消息定义必须重新 `colcon build`；纯 Python 逻辑有时改完 source 就能跑，但 entry_points 变了仍要编译。

新建包：

```bash
ros2 pkg create village_li --build-type ament_python --dependencies rclpy
```

Python 节点要在 `setup.py` 的 `console_scripts` 里登记，否则 `ros2 run` 找不到：

```python
"li4_node = village_li.li4:main"
```

C++ 则是 `add_executable` + `install(TARGETS ... DESTINATION lib/${PROJECT_NAME})`。
