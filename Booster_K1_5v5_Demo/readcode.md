# Booster_K1_5v5_Demo 快速读码指南

这份文档的目标不是逐文件解释所有实现，而是帮你用最短时间建立对这个工作区的整体认知，知道：

- 这个仓库是干什么的
- 主要代码分别在哪
- 运行时数据怎么流
- 应该按什么顺序读
- 遇到某类问题时去哪里找代码

---

## 1. 先建立全局认知

这是一个 RoboCup humanoid soccer 的 ROS 2 `colcon` 工作区，不是一个单体程序，而是多个 ROS 2 包协同运行。

运行时最重要的是 3 个节点：

1. `vision`
   负责视觉感知，做目标检测、分割、测距、线段提取，然后发布感知结果。

2. `game_controller`
   负责接收 RoboCup 裁判机 UDP 广播，转成 ROS topic。

3. `brain`
   负责融合视觉、裁判机、里程计、底层状态，执行行为树决策，再通过机器人 SDK 发动作命令。

如果只记一句话，可以记成：

`vision` 负责看，`game_controller` 负责听裁判，`brain` 负责想和动。

---

## 2. 根目录怎么理解

根目录大致可以分成两类：源码区和构建产物区。

### 2.1 重点看的目录

- `src/`
  所有 ROS 2 包源码都在这里，是主要阅读区。

- `scripts/`
  启停、构建、调试脚本。想知道系统怎么启动，先看这里。

- `distribution/`
  打包和安装脚本，偏部署。

- `configs/`
  一些运行配置，比如 DDS 配置。

### 2.2 阅读时可以先忽略的目录

- `build/`
- `install/`
- `log/`

这三个目录都是构建或运行产物，不是主业务源码。

---

## 3. `src/` 下的包怎么分工

`src/` 下主要有 6 个包值得知道：

### 3.1 `brain`

策略核心包，最重要。

职责：

- 订阅视觉、裁判机、里程计、底层状态
- 维护机器人当前的内部世界模型
- 用 BehaviorTree.CPP 做决策
- 调机器人底层动作接口

### 3.2 `vision`

视觉感知包，代码量最大。

职责：

- 读取相机图像、深度、头部位姿
- 调用 TensorRT/YOLO 模型做检测和分割
- 估计球、标记点、障碍物、线段等位置
- 发布感知结果

### 3.3 `game_controller`

裁判机桥接包。

职责：

- 监听 UDP 端口
- 读取 RoboCup GameController 数据包
- 转成 ROS 消息发布

### 3.4 `booster_ros2_interface`

机器人底层接口定义包。

里面主要是：

- 机器人的消息定义
- 机器人 RPC 服务定义

比如：

- `Odometer.msg`
- `LowState.msg`
- `RemoteControllerState.msg`
- `RpcService.srv`

### 3.5 `robocup_ros2_interface`

比赛相关接口定义包。

里面主要是：

- 裁判机相关消息
- 视觉结果相关消息

比如：

- `GameControlData.msg`
- `Detections.msg`
- `LineSegments.msg`

### 3.6 `booster_msgs`

更底层、更通用的消息包，主要用于 RPC 请求响应封装。

---

## 4. 运行时数据流

系统可以大致看成下面这几条主链路。

### 4.1 视觉链路

`camera + depth + head_pose -> vision -> /booster_vision/* -> brain`

也就是：

- 相机数据进入 `vision`
- `vision` 做识别和估计
- 结果发布到 `/booster_vision/detection`、`/booster_vision/line_segments` 等 topic
- `brain` 再消费这些结果

### 4.2 裁判链路

`GameController UDP -> game_controller -> /robocup/game_controller -> brain`

也就是：

- 裁判机 UDP 包被 `game_controller` 接收
- 转成 ROS 消息
- 发布到 `/robocup/game_controller`
- `brain` 根据比赛状态调整策略

### 4.3 动作链路

`brain -> RobotClient -> LocoApiTopicReq -> 机器人底层执行`

也就是：

- `brain` 决策出要做什么
- `RobotClient` 把动作包装成 SDK / RPC 请求
- 底层控制模块执行动作

---

## 5. 从哪里开始读最省时间

推荐实际阅读顺序如下。

### 第一步：看启动脚本

先看：

- `scripts/start.sh`

你会知道系统实际会拉起哪些节点，以及节点启动顺序。

这个脚本里能看到：

- 先停旧进程
- 再启动 `vision`
- 再启动 `brain`
- 再启动 `game_controller`

这比直接一头扎进源码更容易建立整体感。

### 第二步：看 `brain` 入口

先看：

- `src/brain/src/main.cpp`

这个文件很短，但很关键。它告诉你：

- `Brain` 是主对象
- `brain->init()` 做初始化
- 有一个线程专门做 `brain->tick()`
- 还有一个额外线程处理手柄和裁判机订阅
- 主线程跑 ROS executor

这一步能让你立刻理解 `brain` 的线程模型。

### 第三步：看 `Brain` 类定义

看：

- `src/brain/include/brain.h`

这是理解整个策略包最重要的文件之一。

`Brain` 挂了几个核心对象：

- `BrainConfig`
- `BrainData`
- `BrainLog`
- `BrainTree`
- `BrainCommunication`
- `Locator`
- `RobotClient`

只要看懂这几个成员的职责，整个 `brain` 包基本就不再是黑盒。

### 第四步：看 `brain.cpp`

看：

- `src/brain/src/brain.cpp`

重点先不要试图从头到尾全看完，而是先抓几个关键函数：

- `init()`
- `tick()`
- `gameControlCallback()`
- `detectionsCallback()`
- `fieldLineCallback()`
- `odometerCallback()`
- `lowStateCallback()`
- `updateBallMemory()`
- `updateCostToKick()`

原因很简单：

- `init()` 告诉你这个节点初始化了什么
- `tick()` 告诉你主循环在做什么
- 各种 callback 告诉你数据是怎么进来的
- `update*` 告诉你内部状态是怎么维护的

### 第五步：看行为树 XML

看：

- `src/brain/behavior_trees/game.xml`

这是策略结构的真正入口。

非常重要的一点是：

这个项目里，比赛决策的结构主要不在 C++ 的大 if/else 里，而是在行为树 XML 里。

所以如果你不看 XML，只看 `brain_tree.cpp`，会很容易迷路。

### 第六步：看角色子树

按你关心的角色继续看：

- 前锋：`src/brain/behavior_trees/subtrees/subtree_striker_play.xml`
- 守门员：`src/brain/behavior_trees/subtrees/subtree_goal_keeper_play.xml`

这两个文件能直接告诉你不同角色的主行为节奏。

### 第七步：再回来看动作节点实现

看：

- `src/brain/include/brain_tree.h`
- `src/brain/src/brain_tree.cpp`

这里是行为树节点的具体实现。

建议阅读方式是：

1. 先在 XML 里看到节点名
2. 再回到 `brain_tree.cpp` 搜这个节点名
3. 看它具体怎么读 `brain->data`、怎么调 `brain->client`

这样效率最高。

### 第八步：最后补 `vision` 和消息定义

看：

- `src/vision/include/booster_vision/vision_node.h`
- `src/vision/src/vision_node.cpp`
- `src/robocup_ros2_interface/...`
- `src/booster_ros2_interface/...`

这一步是补齐消息格式和感知细节，不是第一步就该深挖的内容。

---

## 6. `brain` 包怎么理解

### 6.1 它不是普通的“主类 + 一堆工具类”

`brain` 的设计有一个很核心的特点：

很多行为树节点不是只依赖 blackboard，而是直接拿 `Brain*` 指针访问：

- `brain->config`
- `brain->data`
- `brain->client`

所以这里不能简单理解成“纯黑板驱动”的行为树。

更准确地说：

- 行为树负责组织决策结构
- `Brain` 对象负责承载共享状态和执行能力

### 6.2 `BrainConfig`

文件：

- `src/brain/include/brain_config.h`

职责：

- 存静态配置
- 从 YAML 和 launch 参数读取配置
- 生成球场尺寸、相机参数、策略参数等

这里的内容大多是“初始化后基本不变”的值。

比如：

- 球队 ID
- 机器人 ID
- 角色
- 球场尺寸
- 速度上限
- 相机参数
- 避障参数

### 6.3 `BrainData`

文件：

- `src/brain/include/brain_data.h`

职责：

- 存所有运行时动态状态

这是 `brain` 的“内部世界模型”。

里面有：

- 当前球的位置和状态
- 机器人位姿
- 头部角度
- 识别到的障碍物、门柱、场地标记
- 队友通信状态
- 起身状态
- 一些缓存 buffer

如果你想知道“当前机器人脑子里记了什么”，基本都在这里。

### 6.4 `BrainTree`

文件：

- `src/brain/include/brain_tree.h`
- `src/brain/src/brain_tree.cpp`

职责：

- 注册行为树节点
- 加载 XML
- 初始化 blackboard
- 每 tick 执行一次树

它本身更像一个“行为树运行器”。

### 6.5 `RobotClient`

文件：

- `src/brain/include/robot_client.h`
- `src/brain/src/robot_client.cpp`

职责：

- 封装所有对机器人底层动作的调用

比如：

- 走路速度设置
- 转头
- 踢球
- 起身
- 防守动作

可以把它理解成：

`brain` 的唯一动作出口。

如果你想追“某个策略最终怎么发到底层执行”，通常最后都会走到这里。

### 6.6 `BrainCommunication`

文件：

- `src/brain/include/brain_communication.h`
- `src/brain/src/brain_communication.cpp`

职责：

- 向裁判机回 alive 包
- 队友 discovery 广播
- 队友通信单播和接收

也就是多机协作相关逻辑的网络层。

### 6.7 `Locator`

文件：

- `src/brain/include/locator.h`
- `src/brain/src/locator.cpp`

职责：

- 基于场地标记点做粒子滤波定位

如果你关心“机器人怎么知道自己在场上的位置”，主要看这里。

---

## 7. `brain` 主循环在干什么

`Brain::tick()` 的核心逻辑非常短，但含义很重。

大致是：

1. 记录日志和调试信息
2. 更新记忆状态
3. 处理特殊比赛状态
4. 处理多机协作状态
5. `tree->tick()`

也就是说：

- 主循环本身不直接写复杂决策流程
- 决策主干交给行为树
- `tick()` 更像是在做“行为树执行前的状态整理”

这是读这个包时非常重要的认识。

---

## 8. 行为树应该怎么读

### 8.1 入口树 `game.xml`

文件：

- `src/brain/behavior_trees/game.xml`

这个文件先按控制模式分流：

- `control_state == 1`：手动辅助
- `control_state == 2`：重新定位 / 进场
- `control_state == 3`：自动比赛

然后自动比赛模式下，再按裁判状态分流：

- `INITIAL`
- `READY`
- `SET`
- `PLAY`
- `END`
- `FREE_KICK`
- `TIMEOUT`

这相当于整套机器人比赛行为的“总状态机”。

### 8.2 前锋子树

文件：

- `src/brain/behavior_trees/subtrees/subtree_striker_play.xml`

主节奏大致是：

`定位 -> 找球/跟球 -> 决策 -> chase/adjust/kick/cross`

里面最值得注意的是：

- `StrikerDecide`
- `Chase`
- `Adjust`
- `Kick`

这几个节点基本构成了前锋比赛时的主闭环。

### 8.3 守门员子树

文件：

- `src/brain/behavior_trees/subtrees/subtree_goal_keeper_play.xml`

主节奏大致是：

`定位 -> 盯球 -> 决策 -> retreat/chase/adjust/kick/guard`

和前锋相比，守门员更强调：

- 回门前位置
- 守门阻挡位
- 必要时出击

### 8.4 一个很实用的读法

不要一开始就试图从 `brain_tree.cpp` 从上到下读 59 个行为树节点。

更实用的方法是：

1. 在 XML 里看到节点名字
2. 判断这个节点在流程里处于哪一步
3. 去 C++ 里搜索它的实现
4. 看它读了哪些状态、发了什么动作

这样更贴近实际业务流程。

---

## 9. `brain.cpp` 里最值得先看的几个函数

### 9.1 `gameControlCallback`

作用：

- 读取裁判机消息
- 更新行为树 blackboard 中的比赛状态
- 记录我方 / 对方是否开球
- 记录是否罚时
- 同步比分、上下场状态

如果你想知道“裁判状态怎么影响行为树”，先看这里。

### 9.2 `detectionsCallback`

作用：

- 接收视觉检测结果
- 把识别目标按类型分组
- 分成球、门柱、对手、标记点等
- 更新内部世界模型

如果你想知道“球是怎么从视觉进到策略里的”，先看这里。

### 9.3 `fieldLineCallback`

作用：

- 接收视觉识别到的场地线段
- 从机器人坐标系转到球场坐标系
- 做进一步处理和分类

这和定位、场地理解直接相关。

### 9.4 `odometerCallback`

作用：

- 更新里程计坐标
- 维护机器人在场地坐标系中的姿态
- 发布 TF

如果你想知道机器人位姿怎么维护，看这里。

### 9.5 `lowStateCallback`

作用：

- 更新头部 yaw / pitch

这是视觉测距、转头跟踪、头部控制的重要输入。

### 9.6 `updateBallMemory`

作用：

- 维护球的记忆
- 在看不到球时，决定还算不算“知道球在哪”
- 更新球的可用状态

这个函数是“球没看到时系统还能不能继续做判断”的关键。

### 9.7 `updateCostToKick`

作用：

- 计算当前机器人接近并控制球的成本

它主要服务于多机协作决策，比如：

- 谁更该去追球
- 谁更适合成为 lead

---

## 10. `vision` 包怎么读

### 10.1 先看入口

文件：

- `src/vision/src/main.cpp`
- `src/vision/include/booster_vision/vision_node.h`
- `src/vision/src/vision_node.cpp`

这里能看清楚 `vision` 的主流程：

1. 读取配置
2. 初始化相机参数
3. 初始化检测模型
4. 初始化分割模型
5. 初始化姿态估计器
6. 订阅图像和头部姿态
7. 发布检测结果和线段结果

### 10.2 `vision` 内部可以分三层理解

#### 第一层：`base/`

偏基础设施。

比如：

- 数据同步 `DataSyncer`
- 数据记录 `DataLogger`
- 位姿与点云工具

其中 `DataSyncer` 很关键，它负责把：

- 彩色图
- 深度图
- 头部位姿

按时间对齐，供后续推理和位置估计使用。

#### 第二层：`model/`

偏模型推理。

这里主要是：

- YOLOv8 检测器
- YOLOv8 分割器
- TensorRT 推理封装

如果你不是改模型推理本身，第一次读可以先知道有这层，不必先扎进 `.cu` 文件。

#### 第三层：`pose_estimator/`

偏几何计算。

职责：

- 把检测框转成实际空间位置
- 估计球、人体、场地标记的位置
- 拟合场地线段

### 10.3 `vision` 的主要输出

`vision` 主要发布：

- `/booster_vision/detection`
- `/booster_vision/line_segments`
- `/booster_vision/ball`
- `/booster_vision/t_head2base`

这些几乎就是 `brain` 感知输入的主要来源。

---

## 11. `game_controller` 包怎么读

这个包最简单。

核心文件：

- `src/game_controller/src/main.cpp`
- `src/game_controller/include/game_controller_node.h`
- `src/game_controller/src/game_controller_node.cpp`

主要逻辑就是：

1. 建一个 UDP socket
2. 监听 3838 端口
3. 可选做 IP 白名单过滤
4. 把收到的二进制数据包逐字段复制成 ROS 消息
5. 发布到 `/robocup/game_controller`

它基本没有复杂策略逻辑，就是一个协议适配层。

---

## 12. 配置层怎么读

### 12.1 `brain` 配置

主要文件：

- `src/brain/config/config.yaml`
- `src/brain/config/config_local.yaml`（如果存在）
- `src/brain/launch/launch.py`

配置覆盖关系是：

1. `config.yaml`
2. `config_local.yaml`
3. launch 参数

也就是说，越后面的优先级越高。

### 12.2 `vision` 配置

主要文件：

- `src/vision/config/vision.yaml`
- `src/vision/config/vision_local.yaml`（如果存在）
- `src/vision/launch/launch.py`

覆盖逻辑同样是：

1. 默认配置
2. 本地配置
3. 启动时覆盖

### 12.3 配置阅读建议

如果你想知道：

- 当前机器人的默认角色是什么
- 用的是哪种相机
- 模型路径在哪里
- 避障阈值是多少
- 日志是否开启

优先去看 YAML，而不是先在代码里硬搜。

---

## 13. 常见问题应该去哪里找

### 13.1 球是怎么从视觉流进决策的

搜这些关键词：

- `detectionsCallback`
- `ball_location_known`
- `CalcKickDir`
- `StrikerDecide`

### 13.2 定位怎么做

搜这些关键词：

- `Locator`
- `SelfLocate`
- `fieldLineCallback`
- `getMarkersForLocator`

### 13.3 动作怎么发到底层

搜这些关键词：

- `RobotClient`
- `LocoApiTopicReq`
- `setVelocity`
- `kickBall`

### 13.4 多机协作怎么做

搜这些关键词：

- `BrainCommunication`
- `TMStatus`
- `updateCostToKick`
- `isLead`

### 13.5 裁判状态怎么影响行为树

搜这些关键词：

- `gameControlCallback`
- `gc_game_state`
- `gc_game_sub_state`
- `gc_is_kickoff_side`

---

## 14. 真正开始改代码前，最好先建立的几个认识

### 14.1 `Brain` 不是纯协调器

它持有大量核心状态和子系统，不只是个入口对象。

### 14.2 黑板不是唯一状态源

很多 BT 节点直接读写 `brain->config` 和 `brain->data`。

所以排查问题时不要只盯 blackboard。

### 14.3 `BrainData` 是运行时状态真相来源

很多“机器人现在认为什么是真的”都在这里。

### 14.4 行为树 XML 决定结构

如果你只看 C++，很难一下看清整个策略逻辑。

### 14.5 `vision` 输出的是感知结果，不是策略

感知和决策是分开的。

### 14.6 `game_controller` 只做协议转发

比赛规则如何影响行为，主要逻辑仍在 `brain` 里。

---

## 15. 一条最短阅读路线

如果你今天只有 30 到 60 分钟，建议按这个顺序读：

1. `scripts/start.sh`
2. `src/brain/src/main.cpp`
3. `src/brain/include/brain.h`
4. `src/brain/src/brain.cpp` 的 `init()` 和 `tick()`
5. `src/brain/behavior_trees/game.xml`
6. `src/brain/behavior_trees/subtrees/subtree_striker_play.xml`
7. `src/brain/src/brain_tree.cpp` 中 `StrikerDecide`、`Chase`、`Adjust`、`Kick`
8. `src/vision/src/vision_node.cpp`
9. `src/game_controller/src/game_controller_node.cpp`

这样读完以后，你通常已经能回答下面这些问题：

- 系统由哪些节点组成
- 决策主循环在哪里
- 行为树从哪里加载
- 球和线段是怎么进入 `brain` 的
- 动作是怎么发到底层的

---

## 16. 最后一句建议

这个仓库最容易让人迷路的点，不是代码太多，而是：

- 运行时是多节点协作
- `brain` 里既有 blackboard，又有 `BrainData`
- 决策结构在 XML，细节在 C++

所以最有效的读法不是“按文件顺序线性读”，而是：

按“数据流”和“行为流”去追踪。

也就是优先回答这几个问题：

1. 数据从哪里来
2. 数据存到哪里
3. 决策在哪做
4. 动作从哪里发出

只要这 4 个问题打通，这个项目就基本读顺了。
