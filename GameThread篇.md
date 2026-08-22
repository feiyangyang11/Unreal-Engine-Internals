*本系列志在持续更新一些游戏开发相关技术和 Unreal 引擎的原理知识讲解，虽然谈不上细究源码的级别，但已在力求讲清楚 UE 的系统设计以及底层运作模型。对于不甘于在 UE 功能应用层浅尝辄止、正在尝试学习 UE 底层的朋友来说，本系列主要参照本人学习 UE 时的视角去探讨引擎是如何工作的，相信能够在一定程度上帮助你们理解 UE 底层机制以及它是如何支撑应用层的。*

*欢迎阅读该系列文章，并分享自己的理解或提出文章中的模糊、错误的地方（不排除有）。*

*想要阅读系列中其他内容或想要持续关注本系列更新可移步：。*

------

# UE 线程模型深究--Game Thread

## UE 的整体线程模型简介

UE 的线程模型是**以 Game Thread 单线程为主，并搭配众多辅助线程**

Game Thread 负责组织游戏世界，其他线程负责把昂贵、可并行或专用的工作分摊出去

大抵结构如下：

```
                        UE 进程
                           |
        ------------------------------------------------
        |             |            |        |          |
   Game Thread   Render Thread   RHI     Worker     专用线程
                                  Thread  Threads
        |             |                     |
    游戏逻辑        渲染逻辑              通用并行任务
```

接下来先简介每种线程所负责的任务

### Game Thread

UE 进程中只有一条 Game Thread 线程，**用于执行所有核心游戏逻辑**，程序员写的 Gameplay 代码也基本都工作在里面。

例如 Actor Tick、Blueprint Event、输入事件、Timer、委托、RPC函数执行……

可以理解成如下框架：

```
Game Thread
    |
    +-- Input
    +-- Gameplay
    +-- Tick
    +-- Blueprint
    +-- AI
    +-- Timer
    +-- Networking gameplay
    +-- UObject 生命周期
```

### Render Thread

Game Thread 不直接完成渲染，而是**把游戏世界的数据交给 Render Thread 构造渲染指令**，决定游戏世界应该怎么绘制

比如 Game Thread 调用`Character->SetActorLocation(...)`，那么 Render Thread 就获取 Transform、Mesh、Light、Material、Camera 等然后决定画什么、怎么排序、提交哪些 DrawCall……

DrawCall？

### RHI Thread

Rendering Hardware Interface：渲染硬件接口

Render Thread 生成高层渲染指令，而 **RHI 将它们翻译成 GPU 能直接执行的指令，然后提交给 GPU 执行**

### Worker Threads

UE 会建立多个工作线程 Worker Thread，它们**主要负责执行一些复杂的计算任务，目的是将繁重的计算从 Game Thread 中剥离，提高并行能力**

游戏运行过程中系统不断往任务系统 Task Queue 里塞 Task，然后空闲的 Worker Thread 会从里面取出任务执行

比如下面这些任务：

- 动画骨骼计算
- 物理部分计算
- Navigation
- 粒子
- 用户创建的Task
- ……

### 专用 Thread

除了通用 Worker Thread，UE 还会建立一些**专门的 Worker Thread 去处理专门的任务**

- Audio Thread 处理音频
- Async Loading Thread 处理异步加载
- Network 处理网络 IO 
- Physics 相关工作线程
- Slate 相关处理
- 文件 IO 线程

### UE 的并发模型

实际游戏运行时，是**先建立 Game Thread，再由 Game Thread 按需创建和调度各类线程完成 Task**

线程是**执行资源**，Task 是**工作单位**

 UE 的并发模型应该理解成两层：

```
第一层：长期存在的线程
---------------------
Game Thread
Render Thread
RHI Thread
Audio Thread
Worker Thread...

第二层：短生命周期Task
---------------------
Animation Task
Physics Task
Render Task
ParallelFor Task
Gameplay Async Task
...
```

一帧内的工作可以粗略理解为如下（下图中 Game Thread 内的执行顺序未严格区分）

```
Frame N
│
├── Game Thread
│     ├── Input
│     ├── Network Gameplay
│     ├── Timer
│     ├── Actor Tick
│     ├── Blueprint
│     ├── AI
│     └── 更新World
│
├───────── 创建并行Task ───────┐
│                              │
│                         Worker Threads
│                         ├── Animation
│                         ├── Physics
│                         └── Other Tasks
│
├── Render Thread
│     └── 构建Frame N渲染数据
│
├── RHI
│     └── GPU Commands
│
└── GPU
      └── Render
```

## Game Thread

详细分析 Game Thread 如何工作，后面简称为 GT

### 一帧里 Game Thread 怎么跑

```cpp
FEngineLoop::Tick()
        ↓
UGameEngine::Tick()
        ↓
UWorld::Tick()
        │
        ├─ ① 收网络数据 / 处理收到的 RPC、复制数据
        │
        ├─ ② 准备本帧 World Tick
        │
        ├─ ③ TG_PrePhysics
        │      ├─ PlayerController
        │      │    └─ 处理 Gameplay Input
        │      ├─ Actor Tick
        │      ├─ Component Tick
        │      └─ 大量 Blueprint Event Tick
        │
        ├─ ④ TG_StartPhysics
        │
        ├─ ⑤ TG_DuringPhysics
        │
        ├─ ⑥ TG_EndPhysics
        │
        ├─ ⑦ TG_PostPhysics
        │
        ├─ ⑧ Latent Action / Timer / 一些世界级更新
        │
        ├─ ⑨ Camera Update
        │
        ├─ ⑩ TG_PostUpdateWork
        │
        ├─ ⑪ 网络发送/Replication Flush
        │
        └─ ⑫ World Tick 结束
```

#### （1）网络接收

以联网游戏中的服务端的 GT 为例，假设服务端上一帧后接收到了来自客户端发来的数据，到达了网卡/socket

这一帧 `UWorld::Tick()` 在一开始 GT 就会进行网络 Dispatch

即`UWorld::Tick`调用`UNetDriver::TickDispatch`，将 socket 缓冲区中属于本帧的消息（主要是玩家输入和 RPC）读出来并解析，这样后续计算和处理都能依靠最新的状态

但由于网络状况不可预测，所以无法保证本帧的网络消息一定在本帧到达服务器，所以**联网游戏需要预测、插值、校正**

#### （2）物理

Physics（物理）是引擎用来计算物体运动、碰撞、重力、刚体、约束等结果的一套系统

比如一个箱子掉落->受到重力->加速度->落地->碰撞检测->弹起/滚动这一过程，都是**由 UE 的物理系统 Chaos 处理**

在搞懂整个 Physics 过程之前必须理解几个概念

##### TickGroup 与 Physics

因为很多游戏逻辑必须**考虑自己是在“这帧物理计算之前”执行，还是“物理计算完成之后”执行**。比如某帧角色要进行移动，同时要对角色进行射线检测

移动：计算移动意图->物理系统处理碰撞->得到角色最终位置
射线检测：依赖角色最终位置，所以要等待角色完成移动+物理碰撞算完后再进行射线检测

所以，UE 设计了 **PrePhysics、StartPhysics、DuringPhysics、EndPhysics、PostPhysics** 五个阶段，对应着五个 TickGroup 

##### PrimaryActorTick / PrimaryComponentTick 是什么

`PrimaryActorTick / PrimaryComponentTick ` 是一个 actor / component 的**成员变量**，是作为这个 actor / component 的**主 Tick 配置对象**

Tick 是对象实际执行的游戏逻辑，而`PrimaryActorTick / PrimaryComponentTick`**保存这个对象的 Tick 配置，也是对象 Tick 的调度入口**。它管理着这个对象是否开启 tick、tick 属于哪个 tick group、tick 时间间隔、tick 依赖关系……

UE 把这个变量指针注册进 Tick 系统；每帧 Tick 系统调度它时，会调用它的 `ExecuteTick()`，再通过它保存的目标 Actor /  Component找到 `TickActor() / TickComponent()`，最终进入子类重写的 `AActor::Tick() / Component::Tick()`。也就是说，Tick 系统拿到的只是`&PrimaryActorTick / &PrimaryComponentTick`，并且通过它去找到真正的对象并 tick

- `PrimaryActorTick.bCanEverTick`：为 true 则对象开启 tick，否则禁止 tick
- `PrimaryActorTick.TickGroup = TG_PrePhysics` ：对象的 tick 属于 TG_PrePhysics，表示在 PrePhysics 阶段执行该对象的 tick

Actor 类本身有默认的 TickGroup，但代码中也可以手动编码改变。但 TickGroup 并不决定最终执行位置，当 tick 依赖另一个更晚被完成的 tick 时，UE 调度器可能把它延后

#### （3）TG_PrePhysics

它是最常用的 TickGroup 之一。官方把它定义为**一帧开始阶段**，尤其适合需要和物理对象交互的 Actor/Component，把本帧运动和其他会影响物理的状态**准备好**

所有 `TickFunction.TickGroup == TG_PrePhysics` 的 Actor/Component Tick，会在这个阶段被调度。

- PlayerController 接受 PlayerInput 并处理
- Character 和 Movement 更新，如计算加速度 - 计算速度 - 尝试移动角色 - 计算碰撞力
- 大量普通 Actor/Component Tick

#### （4）TG_StartPhysics

前面的 PrePhysics 可能已经产生了很多会影响物理的数据，例如：角色/物体的位置变化、速度变化、碰撞体状态变化……StartPhysics**本身不计算，只是告诉 Chaos 系统数据已备好、可以开始计算**

它的作用更像是 PrePhysics 和 DuringPhysics 之间的**分界线**，标志着本帧 Physics 已经启动

#### （5）TG_DuringPhysics

`TG_DuringPhysics` 是“**物理正在计算期间**”的阶段，允许一些**不依赖最终物理结果**的 Tick 和物理模拟并行进行

- UI 数据更新
- 某些 AI 逻辑
- 不依赖碰撞结果的 Gameplay
- ……

这些**不依赖物理计算结果的游戏逻辑可以在这阶段进行**

需要注意的是，这个阶段 Game Thread 不做物理计算，而是那些不依赖物理结果的逻辑，真正计算物理的是 Chaos Solver （一般工作在其他线程）

#### （6）TG_EndPhysics

`TG_EndPhysics` 和 `TG_StartPhysics` 差不多，主要也是一个**等待/同步完成+分界点**的作用

在这阶段，**收束不依赖最终物理结果的 Tick 和物理模拟计算**这两条并行线，先完成的那条线同步等待另一条线

#### （7）TG_PostPhysics

这一阶段是：本帧物理模拟已经完成之后，**执行那些必须依赖“当前帧最终物理结果”的 Actor / Component Tick**

UE 开始调度那些 `TickGroup = TG_PostPhysics` 的 TickFunction

```
为什么有些 Tick 必须放这里？

假设角色手里有一把枪，只有这一帧角色移动、碰撞、物理相关位置调整完成以后，枪口最终位置才真正稳定，才进行开火 LineTrace，否则射线从旧位置发出，不符合手感且可能出现错位，在高速运动尤甚
```

当然这个例子不是非常恰当，因为开火射线检测不应该放在 tick，而是通常作为定时器事件驱动，但是这也有助于我们理解什么是依赖最终物理结果的 tick 事件

所以我们如果写了某些需要依赖本帧物理计算后的 tick 逻辑，记得将它的主配置设为 TG_PostPhysics

#### （8）Latent Action 

这一阶段，执行 Latent Action ，即**挂起后续执行，等以后条件满足再续上**的事件

UE 的 `FLatentActionManager` 就是专门管理 World 中 pending latent actions 的；它每帧调用 `ProcessLatentActions()` 推进这些 Action

主要以 `Delay()`为例来讲：

```cpp
Event E -> Delay(2.0) -> PrintString(...) -> ……
//第一次执行到这时，可以理解为创建了 FDelayAction：  
RemainingTime = 2.0//延迟到 2 s后执行
CallbackTarget = 当前蓝图对象
ExecutionFunction = 后续恢复入口//恢复后继续从哪执行
UUID = 这个Latent Action的唯一标识
//然后交给 FLatentActionManager 管理
LatentActionManager.AddNewAction(...)
//每帧调用 ProcessLatentActions() 遍历 pending latent actions ，将 Remaining -= DeltaTime，如果到期就恢复执行  
```

#### （9）Timer Manager Tick

这一阶段与 Latent Action 类似，在`FTimerManager`挂起一个**定时事件**

定时事件的主要配置是：**触发时间间隔、触发哪个函数（对象指针+函数指针）、函数参数、是否循环**

定时事件的句柄是 **FTimerHandle**，相当于定时器的 ID

定时器如何使用？

- `GetWorldTimerManager().SetTimer(...)`通过句柄创建定时器并绑定事件、设置参数，并注册到 `FTimerManager`中
- 每帧`FTimerManager`遍历定时器队列看是否有到时间的，到时间就执行绑定的定时事件
- 如果要销毁定时器，调用`ClearTimer(TimerHandle);`

注意，**定时器是帧级定时，所以并非绝对精准**

如果定时器绑定的是 UObject 对象的裸指针，那么并不需要担心生命周期问题，因为 UE 会自动处理。在对象被回收的时候，对应 Timer 会自动取消

#### （10） Camera Update

该阶段计算“**这一帧玩家最终应该从哪里、朝哪个方向、用多大 FOV 看世界**”

玩家视觉依靠核心组件：

- `CameraComponent`：管理相机本身的 transform、FOV 和其他设置
- `PlayerController 的 APlayerCameraManager 组件`：**决定玩家这一帧真正使用哪个 Camera 的视角**

Camera Update 的核心就是要得到 **POV**（Point Of View，玩家视角），包含 location、rotation、FOV，大致流程如下：

```cpp
Camera Update
    ↓
找到当前 ViewTarget //决定使用谁的相机，比如 ViewTarget = Player Character 的 Camera Component
    ↓
获取目标 Camera 信息
    ↓
处理 CameraManager 逻辑
    ↓
处理 Camera Modifier
    ↓
Camera Shake
    ↓
   FOV
    ↓
最终 POV —— Final Camera Transform + Final FOV
//最后具体如何成像的还要去专门研究 MVP 变换
```

#### （11）TG_PostUpdateWork

有些逻辑的 tick 它**依赖这一帧的完整结果状态**，就放在这一阶段去执行

比如枪口特效、屏幕空间相关特效、某些 camera-facing 粒子……

它们不仅依赖物理计算结果、还依赖相机最终位置和方向，所以放在这一帧 tick 的最末尾计算

主要是**粒子系统**

### 为什么这么多逻辑塞在 GT 还能跑得动

虽然 GT 负责**启动、组织、同步**大量 Gameplay 逻辑，但它**本身执行的代码其实很轻**，不涉及复杂的计算

真正昂贵的是：

- 大量路径搜索
- 大量骨骼动画计算
- 大量物理求解
- 大量对象 Spawn/Destroy
- 复杂蓝图
- 大规模遍历
- 资源加载
- 大量复制

UE 大量使用 Task，将动画计算放到 Worker Thread 中，这就是 UE Task Graph 体系，此处不细讲

为避免 GT 过重，禁用不必要的 Actor Tick，低频周期事件将 tick 降频，或改用定时器，或者改用事件驱动

GT 的巨大优势就是少锁甚至无锁，大量 Gameplay 运行在 GT 单线程，大大降低同步成本

### Game Thread 和其他线程怎么交互

主要有三种交互方式：

#### 纯异步计算

GT 将任务和数据**交给 Worker Thread 自行计算**，GT 不关心返回结果

适用于某些统计、日志、无需回传 Gameplay 的后台工作

#### 强制同步

GT 运行到某段逻辑，**需要获取 Worker Thread 的计算结果**，如果未就绪就**同步等待**，类似生产者/消费者

- Tick 中的 DuringPhysics 阶段就是这样，GT 与 physics 双线并行，到 EndPhysics 阶段同步等待收束成一条线
- GT准备动画输入，Worker并行计算Pose，GT继续别的工作，后面需要最终骨骼结果，必须确保动画Task完成

#### 异步回调

Worker Thread 完成计算，需要 GT 完成最后一步，会主动提交一个 GameThread Task 处理这个任务

- Worker Thread 计算完寻路数据后，通知 GT 把路径赋给 AIController / Movement
- Worker Thread 生成大量顶点/高度数据，GT 更新对应 UObject / Component 状态

### Game Thread 常见性能问题

Game Thread 的性能问题，可分成两大类：**GT 自身逻辑过重、GT 被迫同步等待其他任务**

这里只列举一些比较普遍和浅层的问题现象，具体深层原因和优化手段在其他篇章中会深入讲解

##### Tick 太多

同时存在大量：

- Actor Tick
- Component Tick
- Blueprint Event Tick
- Anim Tick
- AI Tick
- Widget Tick

解决方法是能降频就降频，能改成事件驱动就改成事件驱动

##### 单次 Tick 太重

某个 tick 里面进行了大量循环遍历+复杂计算，常见问题：

- 大数组遍历
- 嵌套循环
- 复杂碰撞查询
- 大量 Trace
- 频繁排序
- 频繁查找 Actor
- 大规模距离计算

解决方法是优化算法或者善用 Worker Thread

##### 大量 Spawn / Destroy

```cpp
//一个 Actor Spawn 涉及
分配 UObject
↓
初始化 Actor
↓
创建/初始化/注册 Components
↓
加入 World / Level
↓
Construction
↓
BeginPlay
↓
注册 Tick、注册碰撞、注册渲染代理
↓
可能触发 Replication 等
//-----------------------------------
//一次 Destroy 涉及
Destroy Actor
↓
解绑
↓
注销 Components
↓
移出系统
↓
等待 GC 回收 UObject
```

解决方法是使用**对象池**（游戏开始预先创建 100 个 object，每次要用时从池中取出，用完时归还），尤其适合 Projectile、伤害数字、特效对象、频繁重复的临时Actor……

##### Component 太多导致 Transform 更新过重

RootComponent 移动会导致子 SceneComponent Transform 传播

假设一个角色有 100 个组件，分布在不同层级，角色一次 transform 会带来子组件的大量更新，还附带碰撞、渲染、物理等工作

解决方法是减少冗余组件

##### 碰撞查询 / Trace 太多

碰撞检测和 LineTrace 都是较昂贵的计算，会增加 GT 成本

解决方法是降低检测频率、增加过滤条件（距离远的不检测、不发出声音的不检测……）

##### Blueprint 逻辑过重

蓝图本身跑在虚拟机上执行效率就远低于 C++，如果还包含重逻辑，影响会比较大

解决方法是先减少工作量、再减少执行频率、再优化算法、最后考虑 BP → C++ 热点迁移

##### 网络 Replication / RPC 爆量

服务端 GT 尤其容易遇到

如果大量 Actor 频繁复制属性、频繁 RPC……GT 遍历、检查、序列化、收发容易处理不过来

##### 其他……

还有诸如 GC、缓存不友好、锁竞争、内存分配/拷贝、UMG 逻辑过重……等在此不展开细讲
