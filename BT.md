# 机器人行为树（BT）完整入门指南
> 本文档结合行为树核心节点详解与机器人实战经验，从基础原理到工业级落地，覆盖 **ROS2、Nav2、MoveIt2** 等主流框架，适合机器人开发者从零入门。

---

## 目录
1. [机器人为什么用行为树？](#1-机器人为什么用行为树)
2. [核心运行原理（机器人场景适配）](#2-核心运行原理机器人场景适配)
3. [全节点体系详解（含机器人专用实现）](#3-全节点体系详解含机器人专用实现)
4. [核心机制：黑板与子树](#4-核心机制黑板与子树)
5. [快速上手：ROS2+BehaviorTree.CPP 自主巡逻](#5-快速上手ros2behaviortreecpp-自主巡逻)
6. [工业级最佳实践](#6-工业级最佳实践)
7. [避坑指南](#7-避坑指南)
8. [进阶学习资源](#8-进阶学习资源)

---

## 1. 机器人为什么用行为树？
传统有限状态机（FSM）会随任务复杂度出现“状态爆炸”，而行为树（BT）凭借**层级化、模块化、可视化**特性，已成为 **ROS2 Nav2、MoveIt2** 的官方决策方案。

### 核心优势
| 特性 | 行为树（BT） | 有限状态机（FSM） |
| :--- | :--- | :--- |
| 逻辑结构 | 层级化树形，天然分层 | 扁平化状态+跳转，易混乱 |
| 复用性 | 节点、子树全局复用 | 状态与跳转强耦合，难复用 |
| 可维护性 | 修改局部不影响全局 | 新增状态需改所有关联跳转 |
| 调试难度 | 实时查看节点状态+可视化 | 状态跳转无迹可寻 |
| 硬件安全 | 强制实现`onHalt`终止方法 | 状态中断无统一机制 |

### 典型应用场景
- 自主导航与巡逻（园区、仓库、家庭扫地机器人）
- 机械臂抓取与操作（分拣、装配、服务机器人）
- 多任务协同（导航→识别→抓取→放置→回充）
- 异常处理与容错（低电量回充、导航失败重试）

---

## 2. 核心运行原理（机器人场景适配）
### 2.1 Tick（心跳）机制
- **定义**：行为树的执行触发信号，与机器人**控制周期同步**。
- **频率建议**：
  - 移动机器人（底盘导航）：10–20Hz
  - 机械臂（抓取/操作）：50–100Hz
  - 复合机器人：50Hz
- **关键规则**：
  - Tick必须**非阻塞**，执行时间远小于控制周期（如100Hz周期需<10ms）。
  - 耗时操作（路径规划、物体检测）放**异步线程**，Tick仅检查状态。

### 2.2 三大返回状态
| 状态 | 机器人场景含义 | 典型节点示例 |
| :--- | :--- | :--- |
| `SUCCESS` | 任务完成、条件成立 | 导航到达目标、物体检测成功 |
| `FAILURE` | 任务失败、条件不成立 | 导航超时、未检测到物体 |
| `RUNNING` | 任务进行中，需下一次Tick继续 | 导航移动中、机械臂运动中 |

> ⚠️ **铁律**：
> 1. **Condition节点绝对不能返回RUNNING**。
> 2. **所有持续Action节点必须实现`onHalt`方法**，保证硬件安全。

---

## 3. 全节点体系详解（含机器人专用实现）
行为树由四大核心节点构成，以下结合机器人场景详细说明。



### 3.1 第一大类：根节点（Root Node）
#### 核心定义
整棵树的唯一入口，统一管理生命周期、全局黑板、Tick入口。

#### 严格规则
- 有且仅有1个子节点（多分支需挂控制节点）。
- 无条件Tick子节点，完全继承其返回状态。
- 禁止挂载装饰器。

#### 典型用法
- 根节点下挂`ReactiveSelector`，最左侧处理“急停、低电量”等最高优先级事件。

---

### 3.2 第二大类：控制节点（Composite Node）
#### 3.2.1 序列类节点（Sequence）
核心逻辑：**顺序执行、全成则成、一败则败**。

##### 变体1：带记忆序列（SequenceWithMemory）
- **规则**：子节点返回`RUNNING`时记录位置，下次Tick直接继续。
- **机器人用途**：长流程任务（导航→识别→抓取→放置）。
- **避坑**：不能用于需实时校验前置条件的场景。

##### 变体2：响应式序列（ReactiveSequence）
- **规则**：每次Tick重新执行前置Condition，不满足则中断Action。
- **机器人用途**：安全控制（无障碍物→继续导航）。

#### 3.2.2 选择类节点（Selector/Fallback）
核心逻辑：**优先级执行、一成则成、全败则败**。

##### 变体1：响应式选择器（ReactiveSelector）
- **规则**：每次Tick重新检查高优先级分支，满足则中断低优先级。
- **机器人用途**：异常处理（急停→低电量→导航→待机），**必须放在根节点下**。

#### 3.2.3 并行类节点（Parallel）
- **规则**：同时Tick所有子节点，按`SuccessThreshold`/`FailureThreshold`判断结果。
- **机器人用途**：边移动边检测（谨慎使用，易硬件冲突）。

---

### 3.3 第三大类：执行节点（Execution/Leaf Node）
#### 3.3.1 Condition节点（条件节点）
- **铁律**：只读无副作用、仅返回`SUCCESS`/`FAILURE`、瞬时执行。
- **机器人实现示例（电量检测）**：
```cpp
#include "behaviortree_cpp_v3/behavior_tree.h"
#include "rclcpp/rclcpp.hpp"
#include "sensor_msgs/msg/battery_state.hpp"

class IsBatteryLow : public BT::ConditionNode {
public:
  IsBatteryLow(const std::string& name, const BT::NodeConfiguration& config)
    : BT::ConditionNode(name, config) {
    node_ = rclcpp::Node::make_shared("is_battery_low_node");
    battery_sub_ = node_->create_subscription<sensor_msgs::msg::BatteryState>(
      "/battery_state", 10,
      [this](const sensor_msgs::msg::BatteryState::SharedPtr msg) {
        current_battery_ = msg->percentage;
      });
  }

  static BT::PortsList providedPorts() {
    return { BT::InputPort<double>("threshold", 0.2, "电量阈值（0-1）") };
  }

  BT::NodeStatus tick() override {
    double threshold;
    getInput("threshold", threshold);
    rclcpp::spin_some(node_);
    return (current_battery_ < threshold) ? BT::NodeStatus::SUCCESS : BT::NodeStatus::FAILURE;
  }

private:
  rclcpp::Node::SharedPtr node_;
  rclcpp::Subscription<sensor_msgs::msg::BatteryState>::SharedPtr battery_sub_;
  double current_battery_ = 1.0;
};
```

#### 3.3.2 Action节点（动作节点）
- **完整生命周期**：`setup()`（全局初始化）→ `onInit()`（本次初始化）→ `onRunning()`（核心执行）→ `onHalt()`（强制终止）→ `onTerminate()`（收尾）。
- **机器人实现示例（Nav2导航）**：
```cpp
#include "behaviortree_cpp_v3/behavior_tree.h"
#include "rclcpp/rclcpp.hpp"
#include "nav2_msgs/action/navigate_to_pose.hpp"
#include "rclcpp_action/rclcpp_action.hpp"
#include "geometry_msgs/msg/twist.hpp"

class NavigateToPose : public BT::StatefulActionNode {
public:
  using Nav2Action = nav2_msgs::action::NavigateToPose;
  using GoalHandle = rclcpp_action::ClientGoalHandle<Nav2Action>;

  NavigateToPose(const std::string& name, const BT::NodeConfiguration& config)
    : BT::StatefulActionNode(name, config) {
    node_ = rclcpp::Node::make_shared("navigate_to_pose_node");
    action_client_ = rclcpp_action::create_client<Nav2Action>(node_, "navigate_to_pose");
    cmd_vel_pub_ = node_->create_publisher<geometry_msgs::msg::Twist>("/cmd_vel", 10);
  }

  static BT::PortsList providedPorts() {
    return {
      BT::InputPort<double>("x", "X坐标"),
      BT::InputPort<double>("y", "Y坐标"),
      BT::InputPort<double>("yaw", "偏航角")
    };
  }

  BT::NodeStatus onStart() override {
    double x, y, yaw;
    if (!getInput("x", x) || !getInput("y", y) || !getInput("yaw", yaw)) {
      RCLCPP_ERROR(node_->get_logger(), "目标点参数缺失");
      return BT::NodeStatus::FAILURE;
    }
    if (!action_client_->wait_for_action_server(std::chrono::seconds(5))) {
      RCLCPP_ERROR(node_->get_logger(), "Nav2服务器未启动");
      return BT::NodeStatus::FAILURE;
    }
    auto goal_msg = Nav2Action::Goal();
    goal_msg.pose.header.frame_id = "map";
    goal_msg.pose.pose.position.x = x;
    goal_msg.pose.pose.position.y = y;
    goal_msg.pose.pose.orientation = tf2::toMsg(tf2::Quaternion(tf2::Vector3(0,0,1), yaw));
    auto send_goal_options = rclcpp_action::Client<Nav2Action>::SendGoalOptions();
    send_goal_options.result_callback = [this](const GoalHandle::WrappedResult& result) {
      result_ = result.code;
    };
    goal_handle_future_ = action_client_->async_send_goal(goal_msg, send_goal_options);
    return BT::NodeStatus::RUNNING;
  }

  BT::NodeStatus onRunning() override {
    rclcpp::spin_some(node_);
    if (result_.has_value()) {
      return (*result_ == rclcpp_action::ResultCode::SUCCEEDED) ? 
        BT::NodeStatus::SUCCESS : BT::NodeStatus::FAILURE;
    }
    return BT::NodeStatus::RUNNING;
  }

  void onHalted() override {
    if (auto goal_handle = goal_handle_future_.get()) {
      action_client_->async_cancel_goal(goal_handle);
    }
    geometry_msgs::msg::Twist zero_vel;
    cmd_vel_pub_->publish(zero_vel);
    RCLCPP_WARN(node_->get_logger(), "导航终止，已发送零速指令");
  }

private:
  rclcpp::Node::SharedPtr node_;
  rclcpp_action::Client<Nav2Action>::SharedPtr action_client_;
  rclcpp::Publisher<geometry_msgs::msg::Twist>::SharedPtr cmd_vel_pub_;
  std::shared_future<GoalHandle::SharedPtr> goal_handle_future_;
  std::optional<rclcpp_action::ResultCode> result_;
};
```

---

### 3.4 第四大类：装饰节点（Decorator Node）
#### 机器人必用装饰器
| 装饰器 | 用途 | 必用理由 |
| :--- | :--- | :--- |
| **Timeout** | 给持续Action加超时（如导航超时10秒） | 防止机器人卡死 |
| **Cooldown** | 限制传感器检测频率 | 降低CPU占用 |
| **RepeatUntilSuccess** | 任务失败重试（如重试抓取3次） | 提高容错率 |
| **Inverter** | 条件取反（如`IsNoObstacle`） | 避免重复写Condition |

---

## 4. 核心机制：黑板与子树
### 4.1 黑板（Blackboard）
- **定义**：行为树与ROS2的唯一数据交互通道，键值对容器。
- **数据流向**：
  ```
  ROS2传感器 → 写入黑板 → 行为树决策 → 写入黑板 → ROS2执行器
  ```
- **最佳实践**：
  - ROS2回调只写黑板，不做决策。
  - 键名与ROS2话题名对应（如`/battery_state`→`battery_state`）。
  - 只存小数据（坐标、布尔值），大对象通过话题传递。

### 4.2 子树（Subtree）
- **定义**：把完整行为树封装成节点，实现模块化复用。
- **机器人常用子树**：`Subtree_Navigation`、`Subtree_Grasp`、`Subtree_Recharge`。
- **设计规范**：
  - 单一硬件职责（一个子树只控底盘或机械臂）。
  - 显式输入/输出端口，不直接调用ROS2接口。

---

## 5. 快速上手：ROS2+BehaviorTree.CPP 自主巡逻
### 5.1 环境准备
```bash
# 安装ROS2 Humble（参考官方文档）
# 安装依赖
sudo apt install ros-humble-behaviortree-cpp-v3 ros-humble-navigation2 ros-humble-nav2-bringup
```

### 5.2 创建ROS2包
```bash
cd ~/ros2_ws/src
ros2 pkg create robot_bt_patrol --dependencies rclcpp behaviortree_cpp_v3 nav2_msgs sensor_msgs
cd robot_bt_patrol
mkdir -p src behavior_trees
```

### 5.3 编写节点与XML
- 将前面的`IsBatteryLow`、`NavigateToPose`和`SetPatrolGoal`节点放入`src/robot_bt_nodes.cpp`。
- 在`behavior_trees/patrol_tree.xml`中定义树（参考前文示例）。

### 5.4 编译与运行
```bash
cd ~/ros2_ws
colcon build --packages-select robot_bt_patrol
source install/setup.bash

# 终端1：启动Nav2仿真
ros2 launch nav2_bringup tb3_simulation_launch.py

# 终端2：运行行为树节点
ros2 run robot_bt_patrol robot_bt_patrol_node
```

---

## 6. 工业级最佳实践
### 6.1 节点设计
- 所有控硬件的Action必须实现`onHalt`。
- Condition只读无副作用，耗时操作异步执行。

### 6.2 树结构
- 根节点下挂`ReactiveSelector`，异常处理前置。
- 所有持续Action加`Timeout`。

### 6.3 调试
- 使用**Groot2**实时监控节点状态、黑板数据。
- 每个节点加ROS2日志（`DEBUG`/`INFO`/`WARN`/`ERROR`）。

---

## 7. 避坑指南
| 坑位 | 现象 | 解决方案 |
| :--- | :--- | :--- |
| 导航中断后机器人还在跑 | 未实现`onHalt` | 必须在`onHalt`中发送零速指令 |
| 机器人控制卡顿 | Tick有阻塞操作 | 耗时操作放异步线程 |
| 低电量不回充 | 用了带记忆Selector | 紧急事件用`ReactiveSelector` |
| 导航永远到不了目标 | 用了无记忆Sequence | 持续Action用`SequenceWithMemory` |

---

## 8. 进阶学习资源
- **官方文档**：
  - [BehaviorTree.CPP](https://www.behaviortree.dev/)
  - [Nav2行为树](https://navigation.ros.org/behavior_trees/index.html)
- **开源项目**：
  - [Nav2源码](https://github.com/ros-planning/navigation2)
  - [BehaviorTree.CPP示例](https://github.com/BehaviorTree/BehaviorTree.CPP/tree/master/examples)
- **书籍**：
  - [《Behavior Trees in Robotics and AI》](https://arxiv.org/abs/1709.00084)

---
