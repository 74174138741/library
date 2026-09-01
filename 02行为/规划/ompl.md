# OMPL

[官方文档](https://ompl.kavrakilab.org/) · [Primer PDF](https://ompl.kavrakilab.org/OMPL_Primer.pdf) · [规划器列表](https://ompl.kavrakilab.org/core/planners.html)

OMPL（Open Motion Planning Library）是一套**采样运动规划**库：在状态空间里撒点、连边、避开非法状态，找出从起点到终点的一条可行路径。

它**不是**碰撞检测库，**不是**正逆解，**不是**控制器。机械臂关节限位、和障碍撞不撞、末端能不能到，都要你用回调告诉它。MoveIt、mplib 这类框架把 OMPL 当搜索引擎，自己负责运动学和 FCL 碰撞。

和 ROS 的关系：规划节点常常在 Action 里喊一声「去这个关节目标」，真正搜树的是 OMPL。通信见 [ros2](../../01系统/软件/ros2.md)；MoveIt / Nav2 入口见 [运动规划框架](运动规划框架.md)。

---

## 1. 心智模型

把规划想成在迷宫里找路。OMPL 只认迷宫的**抽象形状**，不认「这是哪款机械臂」。

```
起点 s_start          终点 s_goal
        │                  ▲
        └─► 在状态空间里采样、连线
                 │
                 ▼
            这条边合法吗？ ──► 你提供的 isStateValid()
                 │
                 ▼
            Path（一串状态）──► 可选：简化、插密
```

状态 = 规划变量的一组数。可以是：

- 平面车：$(x,y,\theta)$，空间 `SE2`
- 刚体在三维：位姿 `SE3`
- 机械臂：关节角向量，空间 `RealVector`（再给每轴 bounds）

规划器从不调用 `forwardKinematics`。你在 `isStateValid(state)` 里自己正运动学、自己碰撞检测，返回 true/false。OMPL 只负责「下一次采样谁、边怎么长、树怎么长」。

---

## 2. 每一层在干什么

和 SB3 一样，中间有个导演。OMPL 里导演是 **Planner**；外围是空间、合法性和问题定义。日常用 **SimpleSetup** 把它们捆在一起。

```
你
  │  空间 + 起点终点 + 合法性回调
  ▼
SimpleSetup                 打包层，推荐入口
  ├─ StateSpace             状态长什么样、怎么采样、距离怎么算
  ├─ SpaceInformation       空间 + 合法性 + 检查分辨率
  ├─ ProblemDefinition      起点、终点、优化目标
  ├─ Planner                RRTConnect / RRT* / PRM …
  └─ PathSimplifier         搜完后抄近路、拉直
```

| 层 | 类（常见） | 你要它干什么 |
|---|---|---|
| 状态空间 | `base::SE2StateSpace` `SE3StateSpace` `RealVectorStateSpace` `SO2`/`SO3` | 维度、边界、采样、插值、距离 |
| 空间信息 | `base::SpaceInformation` | 把空间和「合不合法」接上；离散碰撞检查的步长 |
| 合法性 | `StateValidityChecker` 或一个 lambda | **唯一必须你写的核心**：碰撞、限位、约束 |
| 问题 | `base::ProblemDefinition` | 从哪到哪；要可行就行还是要短 |
| 规划器 | `geometric::RRTConnect` 等 | 采样搜索 |
| 路径 | `geometric::PathGeometric` | 状态序列；可 `interpolate`、`simplify` |
| 控制规划 | `control::SimpleSetup` + `ControlSpace` + `StatePropagator` | 不但规划位形，还规划控制量（差速、加速度） |

几何规划（`ompl::geometric`）：路径是状态空间里的折线，默认假设两点之间能直线（或空间自带的 interpolate）走过去。机械臂关节空间规划多半是这一类。

控制规划（`ompl::control`）：还要 `propagate(state, control, duration)`，树按动力学长。底盘有最小转弯半径、无人机有积分器时才需要。

**SimpleSetup 不限制能力**：内部照样创建 `SpaceInformation` 和 `ProblemDefinition`。需要改分辨率、换优化目标时，从 `ss.getSpaceInformation()` 取出来再调。

---

## 3. 一次求解在干什么

```
setStartAndGoal
setStateValidityChecker
setPlanner          （不设则 SimpleSetup 自己挑）
        │
        ▼
solve(time)
        │
        ├─ setup()：算检查分辨率等运行参数
        ├─ 规划器循环：采样 → 往树上长 → 每步问 isStateValid
        └─ 超时或找到路径则停
        │
        ▼
simplifySolution()   可选，缩短折线
getSolutionPath()
```

合法性检查分辨率（`SpaceInformation::setStateValidityCheckingResolution`）是**状态空间最大范围的比例**，不是米。设太粗：边中间可能穿模；设太细：每次碰撞调用爆炸，规划变慢。机械臂关节空间常见从默认（约 1%）再按连杆长度经验调。

采样规划是概率完备：时间够、采样能铺满自由空间，找到路的概率趋向 1。**不保证最优**，除非用带星号的优化规划器（RRT*、PRM*）并且还愿意继续搜。

---

## 4. 规划器怎么选

几何问题默认（你没指定时）：空间有默认投影 → 双向用 **LBKPIECE**，否则 **KPIECE**；没有投影 → 双向 **RRTConnect**，否则 **RRT**。机械臂关节空间有现成 `RealVector`，默认常常落到 KPIECE 家族。实务里机械臂最常用仍是 **RRTConnect**：两棵树对向长，比单树 RRT 快一截。

| 规划器 | 类型 | 什么时候用 |
|---|---|---|
| **RRTConnect** | 双向可行 | 机械臂默认首选，要一条能走的路 |
| RRT | 单向可行 | 目标不好采样、只能从起点长树 |
| **RRT*** | 渐近最优 | 还想路径短，给更多时间继续优化 |
| PRM / PRM* | 路图 | 同一场景多次查询，先建图再反复问 |
| LazyPRM | 路图，边延迟检测 | 碰撞很贵时 |
| KPIECE / BKPIECE / LBKPIECE | 投影格子 | 默认选手；需要还行的投影 |
| EST / BiEST | 树 | 和 RRT 同类，扩展策略不同 |
| SST | 控制/动力学 | 有传播模型时 |

可行 vs 最优：RRTConnect 找到就停，路径往往绕。RRT* 找到第一条后还在rewire。也可以给 RRT* 设「第一条就停」的 termination condition，行为就接近可行规划器。

`range`：每次延伸多远。太大容易穿障、浪费碰撞检测；太小树长得碎。很多封装（包括 mplib）允许你设 `range`，对应 `planner->setRange()`。

---

## 5. 你必须提供的：合法性

OMPL 不内置场景。最小回调：

```cpp
bool isStateValid(const ompl::base::State *state)
{
    // 1. 从 state 取出关节或位姿
    // 2. 正运动学 → 各连杆位姿
    // 3. 碰撞（自身 + 环境），超限位返回 false
    return !collides && within_bounds;
}
```

终点不一定是单个状态：可以是区域、姿态流形、满足约束的集合。OMPL 的 Goal 很泛；若终点根本采样不到，双向树长不起来，只能用单树规划器。

约束规划（末端走直线、保持水平）：用 `ConstrainedSpaceInformation` + 约束流形上的采样，不是在 `isStateValid` 里靠碰运气。mplib 的 OMPL 封装里能看到这条路径。

---

## 6. 和 MoveIt / 自研规划的边界

```
应用（抓取、Nav2）
    │
MoveIt / mplib / 自写节点
    │  FK、碰撞、场景、关节限制、笛卡尔约束
    ▼
OMPL  Planner::solve()
    │  只看见 State 向量和 true/false
    ▼
Path  → 时间参数化 / 轨迹跟踪（不归 OMPL）
```

OMPL 输出的是**几何路径**（状态折线），默认没有时间、速度、加速度。要变成能跟的轨迹，还要 TOPP、迭代时间参数化、或控制器自己插值。不要指望 `solve()` 直接吐 `/joint_trajectory`。

---

## 7. 简单例子（SE2，几何）

平面里从 A 到 B，圆障碍靠回调排除。头文件命名空间习惯：`ob` = `ompl::base`，`og` = `ompl::geometric`。

```cpp
#include <ompl/base/spaces/SE2StateSpace.h>
#include <ompl/geometric/SimpleSetup.h>
#include <ompl/geometric/planners/rrt/RRTConnect.h>

namespace ob = ompl::base;
namespace og = ompl::geometric;

bool isStateValid(const ob::State *state)
{
    const auto *s = state->as<ob::SE2StateSpace::StateType>();
    const double x = s->getX(), y = s->getY();
    const double dx = x - 0.5, dy = y - 0.5;
    return dx * dx + dy * dy > 0.15 * 0.15;  // 避开圆心 (0.5,0.5) 半径 0.15
}

int main()
{
    auto space = std::make_shared<ob::SE2StateSpace>();
    ob::RealVectorBounds bounds(2);
    bounds.setLow(0.0);
    bounds.setHigh(1.0);
    space->setBounds(bounds);

    og::SimpleSetup ss(space);
    ss.setStateValidityChecker(isStateValid);

    ob::ScopedState<ob::SE2StateSpace> start(space), goal(space);
    start->setX(0.1); start->setY(0.1); start->setYaw(0);
    goal->setX(0.9);  goal->setY(0.9);  goal->setYaw(0);
    ss.setStartAndGoalStates(start, goal);

    ss.setPlanner(std::make_shared<og::RRTConnect>(ss.getSpaceInformation()));

    ob::PlannerStatus ok = ss.solve(1.0);  // 最多 1 秒
    if (ok) {
        ss.simplifySolution();
        ss.getSolutionPath().print(std::cout);
    }
}
```

五层对应：

| 代码 | 层 |
|---|---|
| `SE2StateSpace` + `setBounds` | 状态空间 |
| `setStateValidityChecker` | 合法性（你的几何约束） |
| `setStartAndGoalStates` | 问题定义 |
| `RRTConnect` | 规划器 |
| `solve` / `simplifySolution` / `getSolutionPath` | 求解与后处理 |

机械臂把 `SE2` 换成 `RealVectorStateSpace(n)`，bounds 换成各关节限位，`isStateValid` 里做 FK+碰撞，其余骨架不变。

不用 SimpleSetup 时，要自己 `SpaceInformation`、`ProblemDefinition`、`planner->setProblemDefinition(pdef)`、`planner->solve()`。能跑，但更容易漏 `setup()`。官方推荐 SimpleSetup。

---

## 8. 现场怎么查

1. **总失败**：先确认起点、终点自己 `isStateValid` 为 true。终点非法，树永远碰不到。  
2. **很慢**：碰撞回调太贵，或检查分辨率太细，或 `range` 太小。先把回调打日志数次数。  
3. **路径穿模**：分辨率太粗，边中间没查到。加密 `setStateValidityCheckingResolution`，或对解插值后再逐点碰撞。  
4. **路径很绕**：换 RRT* 并加时间，或 `simplifySolution()`（短路 + 缩短）。  
5. **同一地图问很多次**：考虑 PRM 建一次图。  
6. **要跟动力学**：几何规划不够，上 `control::SimpleSetup` 和 `StatePropagator`。

调试时 `ompl::msg::setLogLevel(ompl::msg::LOG_DEBUG)` 能看到采样和失败原因；MoveIt 里对应规划请求的 `planner_id`（如 `RRTConnectkConfigDefault`）。
