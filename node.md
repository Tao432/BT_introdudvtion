# 行为树四大核心节点超详细全解.md
> 本文基于 **BehaviorTree.CPP v3.8+、py_trees v3.0+、Unreal Engine 5** 官方规范编写，覆盖所有工业级节点变体、执行规则、可运行实现、场景用例与避坑红线，是游戏AI、机器人开发的权威参考手册。

---

## 一、四大节点总览与核心设计原则
行为树的所有能力，都由**四大核心节点类型**构成，它们各司其职，共同组成了完整的决策逻辑体系。所有工业级行为树库的节点，都是这四大类的扩展与变体。

### 1.1 四大节点定位与核心特性
| 节点大类 | 核心定位 | 子节点数量 | 是否叶节点 | 核心返回状态规则 |
| :--- | :--- | :--- | :--- | :--- |
| **根节点（Root）** | 行为树唯一入口，生命周期总管 | 有且仅有1个 | 否 | 完全继承唯一子节点的返回状态 |
| **控制节点（Composite）** | 行为树的逻辑骨架，多子节点调度器 | ≥1个 | 否 | 根据子节点执行结果与调度策略，决定最终返回状态 |
| **执行节点（Execution/Leaf）** | 行为树的感官与手脚，唯一与外部系统交互的节点 | 0个 | 是 | Condition仅返回SUCCESS/FAILURE；Action可返回三种状态 |
| **装饰节点（Decorator）** | 行为树的逻辑修饰器，单子节点包装器 | 有且仅有1个 | 否 | 根据装饰逻辑，修改子节点的执行行为与返回结果 |

### 1.2 所有节点通用核心规则
1. **Tick驱动**：所有节点的执行，都必须由父节点的Tick触发，没有Tick，节点不会执行任何逻辑。
2. **深度优先遍历**：行为树默认按**从上到下、从左到右**的深度优先顺序遍历节点。
3. **状态返回**：每个节点执行完成后，必须向父节点返回`SUCCESS`/`FAILURE`/`RUNNING`三种状态之一。
4. **中断机制**：父节点可以随时调用`halt()`方法，强制终止正在`RUNNING`的子节点，子节点必须在`halt()`中完成资源释放与状态重置。
5. **单一职责**：每个节点只负责一件事，避免逻辑耦合，保证复用性与可维护性。

---

## 二、第一大类：根节点（Root Node）
### 2.1 核心定义与设计初衷
根节点是**整棵行为树的唯一入口**，是行为树的顶层节点，每棵行为树有且仅有一个根节点。它的设计初衷是统一管理整棵树的生命周期、全局黑板、Tick入口，为整棵树提供统一的执行入口与状态管理。

### 2.2 严格执行规则
1. **子节点限制**：有且仅有1个子节点，不能有多个子节点。若需要多分支逻辑，必须在根节点下挂载一个控制节点（通常是ReactiveSelector）。
2. **Tick规则**：每次外部触发Tick，根节点会无条件Tick它的唯一子节点，没有任何额外逻辑。
3. **状态返回规则**：根节点的返回状态，完全等于它的唯一子节点的返回状态。
4. **生命周期管理**：根节点负责整棵树的初始化（setup）、运行、终止（shutdown）全生命周期管理。
5. **黑板绑定**：全局黑板通常绑定在根节点上，整棵树的所有节点都可以通过根节点访问全局黑板。
6. **装饰器限制**：绝大多数工业级库（UE、BehaviorTree.CPP）禁止给根节点挂载装饰器节点。

### 2.3 极简实现（C++）
```cpp
#include "behaviortree_cpp_v3/behavior_tree.h"

class RootNode : public BT::TreeNode {
public:
    RootNode(const std::string& name) : TreeNode(name, {}) {}

    // 根节点的Tick逻辑
    BT::NodeStatus tick() override {
        if (child_count() != 1) {
            throw std::runtime_error("根节点必须有且仅有1个子节点");
        }
        // 无条件Tick唯一的子节点
        BT::NodeStatus child_status = children_[0]->executeTick();
        // 完全继承子节点的返回状态
        return child_status;
    }

    BT::NodeType type() const override {
        return BT::NodeType::ROOT;
    }

    // 终止整棵树
    void halt() override {
        haltChild(0);
        resetStatus();
    }
};
```

### 2.4 典型场景用法
- **游戏AI**：根节点下挂载`ReactiveSelector`，最左侧分支处理「角色死亡」最高优先级事件，后续分支处理战斗、巡逻、交互逻辑。
- **机器人开发**：根节点下挂载`ReactiveSelector`，最左侧分支处理「急停、低电量、碰撞检测」紧急安全事件，后续分支处理导航、抓取、回充主任务。

### 2.5 避坑红线
1. ❌ 绝对不能给根节点挂载多个子节点，必须通过控制节点实现多分支。
2. ❌ 绝对不能在根节点里写任何业务逻辑，它只负责转发Tick与状态。
3. ❌ 绝对不能跳过根节点，直接Tick子树，会导致全局生命周期与黑板管理混乱。
4. ❌ 禁止给根节点挂载装饰器，装饰器应挂载在控制节点或执行节点上。

---

## 三、第二大类：控制节点（Composite Node）
控制节点是行为树的**逻辑骨架**，是最核心、最常用的节点类型。它可以拥有多个子节点，核心职责是按照预设的调度策略，决定子节点的执行顺序、中断时机、最终返回结果。

工业级行为树库中，控制节点分为四大子类：**序列类（Sequence）、选择类（Selector/Fallback）、并行类（Parallel）、装饰类控制节点**，下面逐个拆解所有变体的详细规则。

### 3.1 序列类节点（Sequence）
#### 3.1.1 核心定义与设计初衷
序列节点的核心逻辑是**「顺序执行、全成则成、一败则败」**，类比程序里的顺序执行代码，只有前一行代码执行成功，才会执行下一行；任何一行代码报错，整个代码块就报错。

它的设计初衷是实现**有严格先后顺序的线性任务流**，比如「开门→进门→关门」、「导航到目标→识别物体→抓取物体→放置物体」。

---

#### 3.1.2 变体1：基础无记忆序列（Sequence）
##### 严格执行规则
1. 每次Tick，**都会从第一个子节点从头开始执行**，无论上次Tick是否有子节点返回RUNNING。
2. 按从左到右的顺序，依次Tick子节点：
   - 若子节点返回`SUCCESS`，继续Tick下一个子节点。
   - 若子节点返回`FAILURE`，立即终止所有子节点，整体返回`FAILURE`。
   - 若子节点返回`RUNNING`，立即终止本次Tick，整体返回`RUNNING`。
3. 当所有子节点都返回`SUCCESS`，整体返回`SUCCESS`。
4. 节点被`halt()`时，会立即终止所有正在运行的子节点，重置执行状态。

##### 极简实现（Python py_trees）
```python
import py_trees

class Sequence(py_trees.composites.Composite):
    def __init__(self, name="Sequence", memory=False):
        super().__init__(name)
        self.memory = memory
        self.current_child_index = 0

    def tick(self):
        # 无记忆序列：每次Tick都从头开始
        if not self.memory:
            self.current_child_index = 0

        # 从当前索引开始遍历子节点
        for index in range(self.current_child_index, len(self.children)):
            child = self.children[index]
            # Tick子节点
            for status in child.tick():
                yield py_trees.common.Status.RUNNING
            # 处理子节点返回状态
            if child.status == py_trees.common.Status.RUNNING:
                self.current_child_index = index
                self.status = py_trees.common.Status.RUNNING
                return self.status
            elif child.status == py_trees.common.Status.FAILURE:
                self.stop(py_trees.common.Status.FAILURE)
                return self.status
            # SUCCESS则继续下一个子节点

        # 所有子节点都执行成功
        self.stop(py_trees.common.Status.SUCCESS)
        return self.status

    def stop(self, new_status):
        self.current_child_index = 0
        super().stop(new_status)
```

##### 时序执行示例
> 子节点顺序：A（Condition）→ B（Action）→ C（Action）
> - Tick1：A返回SUCCESS → B返回RUNNING → 整体返回RUNNING
> - Tick2：**从头开始**，A返回SUCCESS → B重新被Tick（被重启）→ B返回RUNNING → 整体返回RUNNING
> - 结果：B永远无法执行完成，每次Tick都会被重启

##### 典型场景
- **一次性无状态校验**：「检查门是否关闭→检查门锁是否正常→发送锁门指令」，每次Tick都要重新校验所有前置条件。
- **瞬时任务流**：所有子节点都是瞬时完成的，没有返回RUNNING的节点，比如「读取传感器数据→写入黑板→打印日志」。

##### 避坑红线
❌ **绝对不能用于包含返回RUNNING的持续Action节点的场景**，会导致Action节点每次Tick都被重启，永远无法执行完成。

---

#### 3.1.3 变体2：带记忆序列（SequenceWithMemory/SequenceStar）
这是**90%的序列场景优先使用的节点**，也是工业级项目最常用的序列节点。

##### 严格执行规则
1. 第一次Tick，从第一个子节点开始执行。
2. 按从左到右的顺序，依次Tick子节点：
   - 若子节点返回`SUCCESS`，继续Tick下一个子节点。
   - 若子节点返回`FAILURE`，立即终止所有子节点，**清除记忆**，整体返回`FAILURE`。
   - 若子节点返回`RUNNING`，**记录当前子节点的索引**，立即终止本次Tick，整体返回`RUNNING`。
3. **下一次Tick，直接从上次记录的RUNNING子节点开始执行，不会从头开始**。
4. 当所有子节点都返回`SUCCESS`，**清除记忆**，整体返回`SUCCESS`。
5. 节点被`halt()`时，会立即终止所有正在运行的子节点，清除记忆，重置执行状态。

##### 时序执行示例
> 子节点顺序：A（Condition）→ B（持续Action）→ C（持续Action）
> - Tick1：A返回SUCCESS → B返回RUNNING → 记录索引1 → 整体返回RUNNING
> - Tick2：直接从索引1的B开始Tick → B返回SUCCESS → 继续Tick C → C返回RUNNING → 记录索引2 → 整体返回RUNNING
> - Tick3：直接从索引2的C开始Tick → C返回SUCCESS → 所有子节点完成 → 清除记忆 → 整体返回SUCCESS

##### 典型场景
- **机器人长流程任务**：「导航到目标点→识别物体→抓取物体→放置物体」，导航返回RUNNING时，下次Tick继续执行导航，不会重新开始。
- **游戏AI长动作流**：「走到NPC面前→播放对话动画→触发任务→领取奖励」，动画播放期间不会被重启。

##### 避坑红线
❌ 不能用于需要实时校验前置条件的场景，比如「玩家在视野内→追击玩家」，如果追击过程中玩家消失了，带记忆序列不会重新检查前置条件，会继续执行追击。

---

#### 3.1.4 变体3：响应式序列（ReactiveSequence）
##### 严格执行规则
1. 每次Tick，**都会重新执行所有前面已经返回SUCCESS的Condition节点**，只有所有前置Condition都满足，才会继续执行当前的Action节点。
2. 按从左到右的顺序执行：
   - 若前置Condition节点返回`FAILURE`，立即终止当前正在运行的Action节点，整体返回`FAILURE`。
   - 若所有前置Condition都返回`SUCCESS`，继续Tick当前正在运行的Action节点。
   - 若Action节点返回`SUCCESS`，继续Tick下一个子节点。
   - 若Action节点返回`FAILURE`，整体返回`FAILURE`。
3. 所有子节点都返回`SUCCESS`，整体返回`SUCCESS`。

##### 时序执行示例
> 子节点顺序：A（Condition：玩家在视野内）→ B（Action：追击玩家）
> - Tick1：A返回SUCCESS → B返回RUNNING → 整体返回RUNNING
> - Tick2：**重新执行A** → A返回FAILURE → 立即终止B → 整体返回FAILURE
> - 结果：追击过程中玩家消失，立即停止追击，符合预期

##### 典型场景
- **机器人安全控制**：「无障碍物→继续导航」，导航过程中实时检测障碍物，一旦发现障碍物，立即停止导航。
- **游戏AI战斗逻辑**：「目标存活→持续攻击」，攻击过程中目标死亡，立即停止攻击动作。

##### 避坑红线
❌ 前置Condition节点必须是瞬时、无副作用的，不能有耗时操作，否则会导致Tick卡顿。
❌ 序列中只能有一个持续Action节点，且必须放在所有Condition节点的最后。

---

#### 3.1.5 其他序列变体
| 变体名称 | 核心规则 | 典型用途 |
| :--- | :--- | :--- |
| **IfThenElse** | 三节点序列：Condition→True分支→False分支，Condition成功执行True分支，失败执行False分支 | 简单的条件分支逻辑 |
| **SequenceWithRetry** | 子节点失败后，重试N次，超过次数才整体返回FAILURE | 容错性任务，比如「重试抓取3次」 |
| **SequenceWithTimeout** | 给整个序列设置超时时间，超时则整体返回FAILURE | 有时间限制的任务流 |

---

### 3.2 选择类节点（Selector/Fallback/回退节点）
#### 3.2.1 核心定义与设计初衷
选择节点的核心逻辑是**「优先级执行、一成则成、全败则败」**，类比程序里的`if-else if-else`分支，从左到右优先级从高到低，只要有一个分支执行成功，就停止执行后续分支；只有所有分支都失败，才整体失败。

它的设计初衷是实现**多优先级的分支选择**，比如「优先处理低血量逃跑→不行就攻击→再不行就巡逻」、「优先处理急停事件→再处理导航任务→最后待机」。

---

#### 3.2.2 变体1：基础无记忆选择器（Selector/Fallback）
##### 严格执行规则
1. 每次Tick，**都会从第一个子节点从头开始执行**，无论上次Tick是否有子节点返回RUNNING。
2. 按从左到右的顺序，依次Tick子节点：
   - 若子节点返回`FAILURE`，继续Tick下一个子节点。
   - 若子节点返回`SUCCESS`，立即终止所有子节点，整体返回`SUCCESS`。
   - 若子节点返回`RUNNING`，立即终止本次Tick，整体返回`RUNNING`。
3. 当所有子节点都返回`FAILURE`，整体返回`FAILURE`。
4. 节点被`halt()`时，会立即终止所有正在运行的子节点，重置执行状态。

##### 时序执行示例
> 子节点顺序：A（高优先级：低血量逃跑）→ B（中优先级：攻击）→ C（低优先级：巡逻）
> - Tick1：A返回FAILURE → B返回FAILURE → C返回RUNNING → 整体返回RUNNING
> - Tick2：**从头开始**，A返回SUCCESS → 立即终止C → 整体返回SUCCESS
> - 结果：巡逻过程中血量变低，立即触发高优先级的逃跑逻辑，符合预期

##### 典型场景
- **紧急事件优先处理**：每次Tick都优先检查最高优先级的紧急事件，比如机器人的急停、低电量，游戏AI的死亡、受伤。
- **优先级实时变化的分支选择**：比如「优先攻击近距离敌人→再攻击远距离敌人→最后巡逻」。

##### 避坑红线
❌ 不能用于「一旦选中分支就必须执行到底」的场景，比如「选择一个巡逻点→走到巡逻点」，每次Tick都会重新检查高优先级分支，导致巡逻动作被反复中断。

---

#### 3.2.3 变体2：带记忆选择器（SelectorWithMemory/FallbackStar）
##### 严格执行规则
1. 第一次Tick，从第一个子节点开始执行。
2. 按从左到右的顺序，依次Tick子节点：
   - 若子节点返回`FAILURE`，继续Tick下一个子节点。
   - 若子节点返回`SUCCESS`，立即终止所有子节点，**清除记忆**，整体返回`SUCCESS`。
   - 若子节点返回`RUNNING`，**记录当前子节点的索引**，立即终止本次Tick，整体返回`RUNNING`。
3. **下一次Tick，直接从上次记录的RUNNING子节点开始执行，不会重新检查前面的高优先级分支**。
4. 当所有子节点都返回`FAILURE`，**清除记忆**，整体返回`FAILURE`。
5. 节点被`halt()`时，会立即终止所有正在运行的子节点，清除记忆，重置执行状态。

##### 时序执行示例
> 子节点顺序：A（高优先级）→ B（中优先级）→ C（低优先级）
> - Tick1：A返回FAILURE → B返回FAILURE → C返回RUNNING → 记录索引2 → 整体返回RUNNING
> - Tick2：直接从索引2的C开始Tick → 即使A的条件满足，也不会检查A → C返回SUCCESS → 整体返回SUCCESS
> - 结果：一旦选中C分支，就会执行到底，不会被高优先级分支中断

##### 典型场景
- **一次性任务执行**：「选择一个任务→执行任务直到完成」，比如机器人的「选择一个搬运任务→完成搬运」，执行过程中不会被其他低优先级任务中断。
- **游戏AI的一次性动作**：「选择一个对话NPC→走到NPC面前→完成对话」，执行过程中不会被其他分支中断。

##### 避坑红线
❌ **绝对不能用于紧急事件处理场景**，比如机器人的低电量回充、急停，带记忆选择器不会重新检查高优先级分支，会导致紧急事件无法触发。

---

#### 3.2.4 变体3：响应式选择器（ReactiveSelector）
这是**机器人、游戏AI异常处理的核心节点**，也是工业级项目最常用的选择器节点。

##### 严格执行规则
1. 每次Tick，**都会重新检查所有前面的高优先级分支**，无论当前正在执行哪个低优先级分支。
2. 执行规则：
   - 若高优先级分支的Condition节点返回`SUCCESS`，立即终止当前正在运行的低优先级分支，执行高优先级分支。
   - 若高优先级分支都返回`FAILURE`，继续执行当前正在运行的低优先级分支。
   - 若当前执行的分支返回`SUCCESS`，整体返回`SUCCESS`。
   - 若所有分支都返回`FAILURE`，整体返回`FAILURE`。

##### 时序执行示例
> 子节点顺序：A（最高优先级：急停按钮按下）→ B（高优先级：低电量回充）→ C（中优先级：导航）→ D（低优先级：待机）
> - Tick1：A返回FAILURE → B返回FAILURE → C返回RUNNING → 整体返回RUNNING
> - Tick2：**重新检查A和B** → B返回SUCCESS → 立即终止C → 执行B → B返回RUNNING → 整体返回RUNNING
> - 结果：导航过程中电量变低，立即中断导航，执行回充逻辑，符合安全要求

##### 典型场景
- **机器人安全与异常处理**：根节点下的核心节点，最左侧分支处理急停、碰撞、低电量等最高优先级安全事件，确保任何时候都能及时响应。
- **游戏AI的紧急事件处理**：最左侧分支处理角色死亡、受伤、被攻击等最高优先级事件，确保AI能及时响应突发情况。

##### 避坑红线
❌ 高优先级分支的Condition节点必须是瞬时、无副作用的，不能有耗时操作，否则会导致Tick卡顿。
❌ 高优先级分支不能太多，否则每次Tick都会遍历大量Condition，增加CPU开销。

---

#### 3.2.5 其他选择器变体
| 变体名称 | 核心规则 | 典型用途 |
| :--- | :--- | :--- |
| **RandomSelector** | 每次Tick随机打乱子节点顺序，按选择器规则执行 | NPC随机对话、敌人随机行为，避免AI过于固定 |
| **WeightedSelector** | 给每个子节点设置权重，按权重概率选择子节点执行 | 游戏AI的概率性行为、机器人的随机任务选择 |
| **SwitchSelector** | 根据黑板里的枚举值，选择对应的子节点执行 | 多模式切换，比如机器人的「手动模式/自动模式/调试模式」 |

---

### 3.3 并行类节点（Parallel）
#### 3.3.1 核心定义与设计初衷
并行节点的核心逻辑是**「同时执行所有子节点，按预设阈值判断结果」**，类比程序里的多线程并发执行，不会等待子节点依次执行，而是同时Tick所有子节点。

它的设计初衷是实现**多任务并发执行**，比如机器人的「边移动边检测障碍物」、游戏AI的「边攻击边播放音效」。

#### 3.3.2 严格执行规则
1. 核心参数：
   - `SuccessThreshold`：成功阈值，需要多少个子节点返回`SUCCESS`，整体才返回`SUCCESS`。
   - `FailureThreshold`：失败阈值，需要多少个子节点返回`FAILURE`，整体才返回`FAILURE`。
2. 每次Tick，**都会Tick所有未完成的子节点**，无论子节点是否正在RUNNING。
3. 结果判断规则：
   - 若返回`SUCCESS`的子节点数量 ≥ `SuccessThreshold`，立即终止所有子节点，整体返回`SUCCESS`。
   - 若返回`FAILURE`的子节点数量 ≥ `FailureThreshold`，立即终止所有子节点，整体返回`FAILURE`。
   - 两个阈值都未达到，整体返回`RUNNING`。
4. 节点被`halt()`时，会立即终止所有正在运行的子节点，重置执行状态。

#### 3.3.3 常用并行策略
| 策略名称 | SuccessThreshold | FailureThreshold | 核心规则 | 典型用途 |
| :--- | :--- | :--- | :--- | :--- |
| **全成功才成功（ParallelAll）** | 子节点总数 | 1 | 所有子节点都成功才返回SUCCESS；任一子节点失败就返回FAILURE | 机器人的「边移动边预抓取」，必须移动到位且预抓取完成才算成功 |
| **任一成功就成功（ParallelAny）** | 1 | 子节点总数 | 任一子节点成功就返回SUCCESS；所有子节点失败才返回FAILURE | 多传感器同时检测目标，任一传感器检测到目标就算成功 |
| **全失败才失败（ParallelFailAll）** | 1 | 子节点总数 | 任一子节点成功就返回SUCCESS；所有子节点失败才返回FAILURE | 多路径同时规划，任一规划成功就使用该路径 |

#### 3.3.4 极简实现（C++）
```cpp
#include "behaviortree_cpp_v3/behavior_tree.h"

class ParallelNode : public BT::ControlNode {
public:
    ParallelNode(const std::string& name, int success_threshold, int failure_threshold)
        : ControlNode(name, {}), success_threshold_(success_threshold), failure_threshold_(failure_threshold) {}

    BT::NodeStatus tick() override {
        int success_count = 0;
        int failure_count = 0;

        // 遍历所有子节点，Tick所有未完成的子节点
        for (size_t i = 0; i < children_.size(); i++) {
            auto& child = children_[i];
            if (child->status() == BT::NodeStatus::IDLE || child->status() == BT::NodeStatus::RUNNING) {
                child->executeTick();
            }

            // 统计成功/失败数量
            if (child->status() == BT::NodeStatus::SUCCESS) {
                success_count++;
            } else if (child->status() == BT::NodeStatus::FAILURE) {
                failure_count++;
            }
        }

        // 检查成功阈值
        if (success_count >= success_threshold_) {
            haltChildren();
            return BT::NodeStatus::SUCCESS;
        }
        // 检查失败阈值
        if (failure_count >= failure_threshold_) {
            haltChildren();
            return BT::NodeStatus::FAILURE;
        }
        // 未达到阈值，继续运行
        return BT::NodeStatus::RUNNING;
    }

    void halt() override {
        haltChildren();
        resetStatus();
    }

private:
    int success_threshold_;
    int failure_threshold_;
};
```

#### 3.3.5 典型场景
- **机器人多任务并发**：「底盘导航+激光雷达避障+摄像头物体识别」，三个任务同时执行。
- **游戏AI多动作并发**：「角色移动+攻击动画播放+音效播放+伤害判定」，四个动作同时执行。
- **多传感器融合**：「激光雷达+视觉+超声波」同时检测障碍物，任一传感器检测到障碍物就触发避障。

#### 3.3.6 避坑红线
1. ❌ 非必要不使用并行节点，优先用序列+异步线程实现并发，并行节点易出现硬件资源冲突、逻辑竞争。
2. ❌ 禁止并行执行多个控制同一硬件的节点，比如同时执行两个控制底盘移动的Action，会导致硬件指令冲突。
3. ❌ 并行节点的子节点不能太多，否则每次Tick都会遍历大量子节点，增加CPU开销，影响实时性。
4. ❌ 必须明确设置终止策略，否则会出现子节点一直运行的情况，导致资源泄漏。

---

## 四、第三大类：执行节点（Execution/Leaf Node）
执行节点是行为树的**叶子节点**，没有任何子节点，是行为树唯一能与外部系统交互的节点。它分为两大核心子类：**Condition节点（条件节点）** 和 **Action节点（动作节点）**，是行为树的「感官」和「手脚」。

### 4.1 Condition节点（条件节点）
#### 4.1.1 核心定义与设计初衷
Condition节点是行为树的**「感官」**，核心职责是**只读、无副作用、快速判断环境状态**，告诉行为树「当前的条件是否成立」。

它的设计初衷是实现**环境状态的感知与判断**，为控制节点的分支选择提供依据，比如「玩家是否在视野内」、「电量是否低于阈值」、「是否到达目标点」。

#### 4.1.2 严格执行规则（铁律，违反必出bug）
1. **返回状态限制**：只能返回`SUCCESS`（条件成立）或`FAILURE`（条件不成立），**绝对不能返回`RUNNING`**。
2. **无副作用铁律**：只能做只读操作，绝对不能修改任何系统状态、黑板数据、硬件状态，不能执行任何有实际影响的操作。
3. **瞬时执行铁律**：执行速度必须极快，不能有任何耗时操作、阻塞操作，否则会导致整棵树的Tick卡顿。
4. **无状态铁律**：节点本身不能存储任何状态，所有判断依据都来自黑板或外部系统的只读数据。

#### 4.1.3 完整实现示例（ROS2机器人专用）
```cpp
#include "behaviortree_cpp_v3/behavior_tree.h"
#include "rclcpp/rclcpp.hpp"
#include "sensor_msgs/msg/battery_state.hpp"

// 条件节点：判断机器人电量是否低于阈值
class IsBatteryLow : public BT::ConditionNode {
public:
    IsBatteryLow(const std::string& name, const BT::NodeConfiguration& config)
        : BT::ConditionNode(name, config) {
        // 初始化ROS2节点，订阅电池话题
        node_ = rclcpp::Node::make_shared("is_battery_low_node");
        battery_sub_ = node_->create_subscription<sensor_msgs::msg::BatteryState>(
            "/battery_state", 10,
            [this](const sensor_msgs::msg::BatteryState::SharedPtr msg) {
                // 仅在回调中更新缓存数据，不做任何判断
                current_battery_ = msg->percentage;
            });
    }

    // 定义输入端口：电量阈值
    static BT::PortsList providedPorts() {
        return { BT::InputPort<double>("threshold", 0.2, "电量低阈值（0-1）") };
    }

    // 核心tick函数：仅做只读判断，无任何副作用
    BT::NodeStatus tick() override {
        // 读取输入端口的阈值
        double threshold;
        if (!getInput("threshold", threshold)) {
            threshold = 0.2;
        }
        // 处理ROS2回调，更新电池数据
        rclcpp::spin_some(node_);
        // 只读判断，无任何副作用
        return (current_battery_ < threshold) ? BT::NodeStatus::SUCCESS : BT::NodeStatus::FAILURE;
    }

private:
    rclcpp::Node::SharedPtr node_;
    rclcpp::Subscription<sensor_msgs::msg::BatteryState>::SharedPtr battery_sub_;
    double current_battery_ = 1.0; // 仅缓存数据，不做修改
};
```

#### 4.1.4 典型场景
- **机器人场景**：`IsObstacleDetected`（是否检测到障碍物）、`IsGoalReached`（是否到达目标点）、`IsEStopPressed`（急停按钮是否按下）。
- **游戏AI场景**：`IsPlayerInSight`（玩家是否在视野内）、`IsHealthLow`（血量是否过低）、`IsTargetAlive`（目标是否存活）。

#### 4.1.5 避坑红线（新手90%的bug来源）
1. ❌ 绝对不能在Condition节点里写有副作用的动作，比如「开门」、「发送指令」、「修改黑板数据」，会导致每次Tick都执行动作，逻辑混乱。
2. ❌ 绝对不能返回`RUNNING`，会导致父节点的执行逻辑完全错乱。
3. ❌ 绝对不能有耗时操作，比如同步路径规划、模型推理、阻塞式服务调用，会导致Tick卡顿，影响机器人控制实时性。
4. ❌ 绝对不能在Condition节点里初始化硬件、创建ROS2订阅者/发布者，初始化操作必须放在构造函数里。

---

### 4.2 Action节点（动作节点）
#### 4.2.1 核心定义与设计初衷
Action节点是行为树的**「手脚」**，核心职责是**执行具体的行为、改变系统状态、与硬件/外部系统交互**，是行为树唯一能产生实际动作的节点。

它的设计初衷是实现**具体的行为执行**，比如机器人的「导航到目标点」、「抓取物体」，游戏AI的「移动到位置」、「播放动画」、「攻击目标」。

#### 4.2.2 完整生命周期（工业级规范，必须严格遵守）
Action节点有完整的5段生命周期，所有工业级库都遵循这个规范，这是保证机器人硬件安全、逻辑正确的核心。

| 生命周期方法 | 调用时机 | 核心职责 | 必须实现的操作 |
| :--- | :--- | :--- | :--- |
| **setup()** | 行为树初始化时，仅执行一次 | 全局初始化，打开硬件连接、创建ROS2节点/订阅者/发布者、加载模型 | 硬件初始化、资源申请、全局配置加载 |
| **onInit()/initialise()** | 节点从IDLE状态切换到RUNNING状态时，每次启动执行一次 | 本次执行的初始化，从黑板读取参数、初始化执行环境、准备动作指令 | 读取输入参数、初始化动作状态、准备执行指令 |
| **onRunning()/update()** | 每次Tick都会调用，核心执行逻辑 | 执行动作的核心逻辑，检查动作执行状态，返回当前状态 | 更新动作状态、检查执行结果、返回SUCCESS/FAILURE/RUNNING |
| **onHalt()/stop()** | 节点被父节点强制终止时调用，必须实现 | 停止动作、释放资源、重置状态，保证硬件安全 | 立即停止硬件运动、取消异步任务、释放资源、重置状态 |
| **onTerminate()/terminate()** | 节点从RUNNING状态切换到SUCCESS/FAILURE状态时调用 | 执行完成后的收尾工作，清理临时数据、记录日志 | 清理临时数据、记录执行结果、释放临时资源 |

#### 4.2.3 Action节点三大分类与实现
根据执行时长与执行方式，Action节点分为三大类，分别适用于不同场景。

---

##### 分类1：同步瞬时Action（SyncActionNode）
- 核心特性：一帧内完成执行，无持续过程，不会返回`RUNNING`。
- 严格规则：只能返回`SUCCESS`或`FAILURE`，不能返回`RUNNING`，无需实现`onHalt()`。
- 典型用途：设置黑板变量、发送瞬时指令、记录日志、播放短音效。
- 实现示例（C++）：
```cpp
#include "behaviortree_cpp_v3/behavior_tree.h"
#include "geometry_msgs/msg/pose_stamped.hpp"

// 同步瞬时Action：设置黑板里的目标点
class SetTargetPose : public BT::SyncActionNode {
public:
    SetTargetPose(const std::string& name, const BT::NodeConfiguration& config)
        : BT::SyncActionNode(name, config) {}

    static BT::PortsList providedPorts() {
        return {
            BT::InputPort<double>("x", "目标点X坐标"),
            BT::InputPort<double>("y", "目标点Y坐标"),
            BT::OutputPort<geometry_msgs::msg::PoseStamped>("target_pose", "输出目标位姿")
        };
    }

    // 同步执行，一帧完成，只能返回SUCCESS/FAILURE
    BT::NodeStatus tick() override {
        double x, y;
        if (!getInput("x", x) || !getInput("y", y)) {
            RCLCPP_ERROR(rclcpp::get_logger("SetTargetPose"), "目标点参数缺失");
            return BT::NodeStatus::FAILURE;
        }
        // 构建目标位姿
        geometry_msgs::msg::PoseStamped pose;
        pose.header.frame_id = "map";
        pose.pose.position.x = x;
        pose.pose.position.y = y;
        // 写入黑板
        setOutput("target_pose", pose);
        return BT::NodeStatus::SUCCESS;
    }
};
```

---

##### 分类2：状态持续Action（StatefulActionNode）
- 核心特性：跨多Tick执行，有持续过程，会返回`RUNNING`，是最常用的Action节点类型。
- 严格规则：必须实现完整的生命周期，特别是`onHalt()`方法，保证节点被终止时能安全停止动作。
- 典型用途：机器人导航、机械臂运动、动画播放、长时任务执行。
- 实现示例（ROS2 Nav2导航专用）：
```cpp
#include "behaviortree_cpp_v3/behavior_tree.h"
#include "rclcpp/rclcpp.hpp"
#include "nav2_msgs/action/navigate_to_pose.hpp"
#include "rclcpp_action/rclcpp_action.hpp"
#include "tf2_geometry_msgs/tf2_geometry_msgs.hpp"
#include "geometry_msgs/msg/twist.hpp"

// 状态持续Action：导航到目标点
class NavigateToPose : public BT::StatefulActionNode {
public:
    using Nav2Action = nav2_msgs::action::NavigateToPose;
    using GoalHandle = rclcpp_action::ClientGoalHandle<Nav2Action>;

    NavigateToPose(const std::string& name, const BT::NodeConfiguration& config)
        : BT::StatefulActionNode(name, config) {
        // setup()阶段：创建ROS2节点和动作客户端，仅执行一次
        node_ = rclcpp::Node::make_shared("navigate_to_pose_node");
        action_client_ = rclcpp_action::create_client<Nav2Action>(node_, "navigate_to_pose");
    }

    static BT::PortsList providedPorts() {
        return {
            BT::InputPort<geometry_msgs::msg::PoseStamped>("target_pose", "导航目标位姿"),
            BT::OutputPort<bool>("navigation_success", "导航是否成功")
        };
    }

    // onInit()：每次启动执行时调用，仅执行一次
    BT::NodeStatus onStart() override {
        // 1. 读取输入参数
        geometry_msgs::msg::PoseStamped target_pose;
        if (!getInput("target_pose", target_pose)) {
            RCLCPP_ERROR(node_->get_logger(), "导航目标位姿缺失");
            return BT::NodeStatus::FAILURE;
        }
        // 2. 检查动作服务器是否可用
        if (!action_client_->wait_for_action_server(std::chrono::seconds(5))) {
            RCLCPP_ERROR(node_->get_logger(), "Nav2动作服务器未启动");
            return BT::NodeStatus::FAILURE;
        }
        // 3. 构建导航目标，发送异步请求
        auto goal_msg = Nav2Action::Goal();
        goal_msg.pose = target_pose;
        auto send_goal_options = rclcpp_action::Client<Nav2Action>::SendGoalOptions();
        send_goal_options.result_callback = [this](const GoalHandle::WrappedResult& result) {
            result_code_ = result.code;
        };
        goal_handle_future_ = action_client_->async_send_goal(goal_msg, send_goal_options);
        // 4. 返回RUNNING，等待导航完成
        RCLCPP_INFO(node_->get_logger(), "已发送导航目标，开始导航");
        return BT::NodeStatus::RUNNING;
    }

    // onRunning()：每次Tick都会调用，检查导航状态
    BT::NodeStatus onRunning() override {
        // 处理ROS2回调
        rclcpp::spin_some(node_);
        // 检查导航结果
        if (result_code_.has_value()) {
            bool success = (*result_code_ == rclcpp_action::ResultCode::SUCCEEDED);
            setOutput("navigation_success", success);
            RCLCPP_INFO(node_->get_logger(), "导航完成，结果：%s", success ? "成功" : "失败");
            return success ? BT::NodeStatus::SUCCESS : BT::NodeStatus::FAILURE;
        }
        // 导航仍在进行中
        return BT::NodeStatus::RUNNING;
    }

    // onHalt()：节点被强制终止时调用，必须实现！保证硬件安全
    void onHalted() override {
        // 1. 取消导航目标
        if (goal_handle_future_.valid()) {
            auto goal_handle = goal_handle_future_.get();
            if (goal_handle) {
                action_client_->async_cancel_goal(goal_handle);
                RCLCPP_WARN(node_->get_logger(), "导航被强制终止，已发送取消指令");
            }
        }
        // 2. 重置状态
        result_code_.reset();
        // 3. 【关键】给底盘发送零速指令，停止机器人移动
        auto cmd_vel_pub = node_->create_publisher<geometry_msgs::msg::Twist>("/cmd_vel", 10);
        geometry_msgs::msg::Twist zero_vel;
        cmd_vel_pub->publish(zero_vel);
        RCLCPP_WARN(node_->get_logger(), "已发送零速指令，停止底盘运动");
    }

private:
    rclcpp::Node::SharedPtr node_;
    rclcpp_action::Client<Nav2Action>::SharedPtr action_client_;
    std::shared_future<GoalHandle::SharedPtr> goal_handle_future_;
    std::optional<rclcpp_action::ResultCode> result_code_;
};
```

---

##### 分类3：异步线程Action（ThreadedActionNode）
- 核心特性：把耗时操作放在独立的异步线程中执行，Tick里只检查线程执行状态，不会阻塞主Tick循环。
- 严格规则：耗时操作必须放在独立线程中，线程间数据交互必须线程安全，`onHalt()`必须能安全终止线程。
- 典型用途：物体检测模型推理、路径规划、文件读写、网络请求等耗时操作。

#### 4.2.4 避坑红线（机器人硬件安全第一）
1. ❌ **所有控制硬件的持续Action节点，必须实现`onHalt()`方法**，节点被终止时必须立即停止硬件运动，否则会导致机器人失控、硬件损坏。
2. ❌ 绝对不能在`update()`/`onRunning()`方法里写阻塞操作，必须把耗时操作放在异步线程中，否则会导致主Tick循环卡顿，影响机器人控制实时性。
3. ❌ 绝对不能在`onInit()`方法里写耗时操作，`onInit()`必须瞬时完成，否则会导致行为树执行卡顿。
4. ❌ 节点执行完成后，必须清理临时数据、重置状态，避免下次执行时出现状态残留。
5. ❌ 禁止在Action节点里直接调用其他节点的方法，所有数据交互都必须通过黑板，保证节点解耦。

---

## 五、第四大类：装饰节点（Decorator Node）
### 5.1 核心定义与设计初衷
装饰节点是行为树的**「逻辑修饰器」**，它有且仅有1个子节点，核心职责是**修改子节点的执行行为、返回结果、执行次数、执行时机**，无需修改子节点本身，就能实现逻辑的灵活扩展，类比程序里的「装饰器模式」。

它的设计初衷是实现**通用逻辑的复用**，比如超时控制、重试、结果反转、冷却等通用逻辑，只需要写一个装饰节点，就能给所有子节点复用，无需在每个Action节点里重复编写。

### 5.2 装饰节点通用执行规则
1. **子节点限制**：有且仅有1个子节点，不能有多个子节点。
2. **Tick转发**：装饰节点收到Tick后，根据装饰逻辑，决定是否Tick子节点、何时Tick子节点。
3. **状态修改**：装饰节点可以修改子节点的返回状态，决定最终向父节点返回的状态。
4. **中断转发**：装饰节点被`halt()`时，必须立即`halt()`它的子节点，保证子节点被正确终止。

### 5.3 常用装饰节点全解（含机器人场景用法）
下面按功能分类，拆解所有工业级项目常用的装饰节点，每个节点都包含详细规则、用例、避坑红线。

---

#### 5.3.1 结果修改类装饰节点
这类装饰节点的核心作用是**修改子节点的返回结果**，无需修改子节点本身的逻辑。

##### 1. Inverter（反转节点/取反节点）
- **严格执行规则**：
  1. Tick子节点，获取子节点的返回状态。
  2. 若子节点返回`SUCCESS`，整体返回`FAILURE`。
  3. 若子节点返回`FAILURE`，整体返回`SUCCESS`。
  4. 若子节点返回`RUNNING`，整体返回`RUNNING`，不做修改。
- **典型用途**：取反条件判断，比如`IsPlayerNotInSight = Inverter + IsPlayerInSight`，无需重复编写条件节点；机器人场景中`IsNoObstacle = Inverter + IsObstacleDetected`。
- **避坑红线**：只能用于Condition节点或瞬时Action节点，不能用于持续Action节点，否则会导致逻辑混乱。

##### 2. ForceSuccess（强制成功节点）
- **严格执行规则**：
  1. Tick子节点，直到子节点返回`SUCCESS`/`FAILURE`。
  2. 无论子节点返回`SUCCESS`还是`FAILURE`，整体都返回`SUCCESS`。
  3. 子节点返回`RUNNING`时，整体返回`RUNNING`。
- **典型用途**：不影响主流程的分支，比如机器人的「播放提示音」、「记录日志」，即使执行失败，也不中断主任务。
- **避坑红线**：不能用于核心任务节点，否则会掩盖任务失败的问题，导致逻辑错误。

##### 3. ForceFailure（强制失败节点）
- **严格执行规则**：
  1. Tick子节点，直到子节点返回`SUCCESS`/`FAILURE`。
  2. 无论子节点返回`SUCCESS`还是`FAILURE`，整体都返回`FAILURE`。
  3. 子节点返回`RUNNING`时，整体返回`RUNNING`。
- **典型用途**：临时禁用某个分支，调试时使用，无需删除节点；异常测试时，强制某个分支失败，测试容错逻辑。
- **避坑红线**：不能用于生产环境的核心逻辑，否则会导致任务永远失败。

---

#### 5.3.2 执行控制类装饰节点（机器人必用）
这类装饰节点的核心作用是**控制子节点的执行时机、执行次数、执行时长**，是机器人项目中最常用的装饰节点。

##### 1. Timeout（超时节点，机器人必用）
- **核心定义**：给子节点设置最大执行时长，超时则强制终止子节点，返回`FAILURE`，是防止机器人卡死的核心节点。
- **严格执行规则**：
  1. 第一次Tick子节点时，启动计时器。
  2. 每次Tick都会检查计时器是否超过设定的超时时间。
  3. 若子节点在超时时间内返回`SUCCESS`/`FAILURE`，停止计时器，整体返回子节点的结果。
  4. 若超过超时时间，子节点仍在`RUNNING`，立即`halt()`子节点，整体返回`FAILURE`。
- **典型用途**：给所有持续Action节点加超时控制，比如「导航超时10秒失败」、「抓取超时5秒失败」、「模型推理超时2秒失败」，防止节点一直返回RUNNING，整棵树卡死。
- **避坑红线**：
  - ❌ 所有持续Action节点必须加Timeout装饰器，这是机器人项目的安全铁律。
  - ❌ 超时时间必须合理，不能过短（导致任务还没完成就被终止），也不能过长（导致卡死时间太久）。

##### 2. Cooldown（冷却节点）
- **核心定义**：限制子节点的执行频率，子节点执行完成后，进入冷却时间，冷却时间内再次触发，直接返回`FAILURE`，不执行子节点。
- **严格执行规则**：
  1. 冷却时间未结束时，收到Tick直接返回`FAILURE`，不执行子节点。
  2. 冷却时间结束后，收到Tick会执行子节点。
  3. 子节点执行完成（返回`SUCCESS`/`FAILURE`）后，重新启动冷却计时器。
  4. 子节点返回`RUNNING`时，不启动冷却计时器，直到子节点执行完成。
- **典型用途**：限制传感器检测频率，比如「物体检测冷却100ms，避免高频调用摄像头」；限制动作执行频率，比如「攻击动作冷却2秒」、「语音播报冷却5秒」。
- **避坑红线**：冷却时间必须大于子节点的最大执行时长，否则会导致子节点还没执行完成，冷却时间就结束了。

##### 3. Delay（延迟执行节点）
- **严格执行规则**：
  1. 第一次收到Tick时，启动延迟计时器，返回`RUNNING`。
  2. 延迟时间内，每次Tick都返回`RUNNING`，不执行子节点。
  3. 延迟时间结束后，开始Tick子节点，返回子节点的执行状态。
  4. 节点被`halt()`时，停止计时器，终止子节点。
- **典型用途**：优化动作时序，比如机器人「到达目标点后，延迟1秒再执行抓取，避免底盘抖动影响抓取精度」；游戏AI「看到玩家后，延迟0.5秒再说话，更自然」。
- **避坑红线**：延迟时间不能过长，否则会影响行为树的响应速度。

##### 4. Repeat（重复执行节点）
- **严格执行规则**：
  1. 重复执行子节点N次，N为设定的循环次数，-1表示无限循环。
  2. 子节点返回`SUCCESS`，计数+1，继续执行下一次循环。
  3. 子节点返回`FAILURE`，立即终止循环，整体返回`FAILURE`。
  4. 子节点返回`RUNNING`，整体返回`RUNNING`。
  5. 达到循环次数上限后，整体返回`SUCCESS`。
- **典型用途**：重复执行某个动作，比如「巡逻3圈」、「重试抓取3次」、「循环播放巡逻动画」。
- **避坑红线**：无限循环必须设置终止条件，否则会导致死循环，整棵树卡死。

##### 5. RepeatUntilSuccess（重复直到成功）
- **严格执行规则**：
  1. 重复执行子节点，直到子节点返回`SUCCESS`，整体返回`SUCCESS`。
  2. 可设置最大重试次数，超过次数后，整体返回`FAILURE`。
  3. 子节点返回`RUNNING`，整体返回`RUNNING`。
- **典型用途**：容错处理，比如「重试连接设备，直到成功」、「重试抓取物品，最多5次」、「重试导航，最多3次」。
- **避坑红线**：必须设置最大重试次数，不能无限重试，否则会导致机器人卡在某个任务上永远无法退出。

---

#### 5.3.3 状态管理类装饰节点
##### 1. Once（只执行一次节点）
- **严格执行规则**：
  1. 子节点只会被执行一次，无论结果是`SUCCESS`还是`FAILURE`。
  2. 第一次执行完成后，之后每次收到Tick，都直接返回上次的执行结果，不会再执行子节点。
  3. 节点被`halt()`时，不会重置执行状态，只有整棵树重置时才会重置。
- **典型用途**：只需要执行一次的初始化动作，比如机器人「开机时初始化硬件」、「启动时加载配置文件」；游戏AI「游戏开始时加载角色数据」。
- **避坑红线**：不能用于需要重复执行的任务，否则会导致任务只执行一次就不再执行。

##### 2. KeepRunningUntilFailure（保持运行直到失败）
- **严格执行规则**：
  1. 每次Tick都会执行子节点。
  2. 子节点返回`SUCCESS`时，整体返回`RUNNING`，继续执行子节点。
  3. 子节点返回`FAILURE`时，整体返回`FAILURE`，停止执行。
  4. 子节点返回`RUNNING`时，整体返回`RUNNING`。
- **典型用途**：循环执行某个任务，直到触发失败条件，比如「循环巡逻，直到发现玩家」、「循环检测障碍物，直到检测到障碍物」。
- **避坑红线**：必须设置明确的失败条件，否则会导致死循环。

---

## 六、四大节点选型速查表
| 需求场景 | 首选节点类型 | 不推荐节点类型 |
| :--- | :--- | :--- |
| 严格顺序执行的线性任务流 | SequenceWithMemory | 基础无记忆Sequence |
| 紧急事件优先处理、异常处理 | ReactiveSelector | SelectorWithMemory |
| 需要实时校验前置条件的任务 | ReactiveSequence | SequenceWithMemory |
| 一次性无状态校验任务 | 基础无记忆Sequence | SequenceWithMemory |
| 一旦选中分支就执行到底的任务 | SelectorWithMemory | ReactiveSelector |
| 多任务并发执行 | Parallel（谨慎使用） | 序列+同步阻塞 |
| 环境状态判断、条件检查 | Condition节点 | Action节点 |
| 硬件控制、动作执行 | StatefulActionNode | SyncActionNode |
| 防止任务卡死、超时控制 | Timeout装饰器 | 无超时的Action节点 |
| 限制动作执行频率 | Cooldown装饰器 | 节点内硬编码延迟 |
| 任务失败重试 | RepeatUntilSuccess装饰器 | 节点内循环重试 |
| 条件取反 | Inverter装饰器 | 重复编写Condition节点 |

---

## 七、四大节点通用避坑铁律
1. **根节点**：只能有一个子节点，不写业务逻辑，不挂载装饰器。
2. **控制节点**：左高右低原则，Selector分支优先级从左到右依次降低；优先使用带记忆/响应式变体，避免基础无记忆变体；非必要不使用Parallel节点。
3. **执行节点**：Condition节点只读无副作用，不返回RUNNING；Action节点必须实现完整生命周期，特别是`onHalt()`方法，保证硬件安全；耗时操作必须异步执行，不阻塞主Tick。
4. **装饰节点**：优先使用通用装饰器实现通用逻辑，避免在Action节点里重复编写；所有持续Action必须加Timeout装饰器；禁止多层嵌套装饰器，否则会导致逻辑难以调试。