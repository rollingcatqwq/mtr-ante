# MTR-ANTE 技术架构与代码深度分析报告

本文档提供了 **MTR-ANTE (Aphrodite's Nemo's Transit Expansion)** 项目的详细技术分析。该项目是 Minecraft Transit Railway (MTR) 模组的一个扩展，通过引入自定义渲染引擎、脚本系统和高级建造工具，极大地增强了 MTR 的功能。

## 1. 核心架构概览

MTR-ANTE 的架构可以分为以下几个核心层次：

1.  **渲染层 (Rendering Layer)**: 基于自研的 **Sowcer** 引擎，接管了 MTR 的轨道渲染，实现了高性能的实例化渲染 (Instanced Rendering)。
2.  **脚本层 (Scripting Layer)**: 集成 **GraalVM Polyglot**，提供了一个安全的 JavaScript 沙盒环境，允许资源包通过脚本控制 3D 模型和交互逻辑。
3.  **数据层 (Data Layer)**: 采用完全的数据驱动模式，通过 JSON 配置定义轨道模型、Eye Candy（装饰物）属性等。
4.  **交互层 (Interaction Layer)**: 提供了多种高级工具（如复合建造器、路径编辑器），直接操作 MTR 的底层数据结构。
5.  **集成层 (Integration Layer)**: 通过 SpongePowered Mixin 深度注入 MTR 代码，修改其默认行为。

---

## 2. 模块详细分析

### 2.1 Sowcer 渲染引擎 (`cn.zbx1425.sowcer`)

Sowcer (Simple OpenGL Wrapper for Complex Entity Rendering) 是项目的渲染核心。

*   **`ContextCapability.java`**: 检测当前 OpenGL 上下文的能力（如是否支持实例化渲染），为渲染器选择降级策略提供依据。
*   **`batch/BatchManager.java`**: 渲染批次管理器。负责收集来自不同来源的绘制请求，尽量合并 Draw Call 以提高性能。
*   **`batch/EnqueueProp.java`**: 定义了加入批次时的属性（如渲染状态）。
*   **`batch/ShaderProp.java`**: 管理着色器属性，如光照贴图纹理 ID、覆盖层纹理 ID 等。
*   **`object/InstanceBuf.java`**: 封装 OpenGL 的 Instance Buffer 对象，用于存储每个实例的数据（如变换矩阵、颜色）。
*   **`object/VertBuf.java`**: 封装 OpenGL 的 Vertex Buffer 对象。
*   **`util/OffHeapAllocator.java`**: 简单的堆外内存分配器，用于高效传输数据到显存。
*   **`vertex/VertAttrMapping.java`**: 定义顶点属性的内存布局（Stride, Offset, Type）。

### 2.2 渲染实现 (`cn.zbx1425.mtrsteamloco.render`)

该包将 Sowcer 应用于具体的游戏对象渲染。

*   **`RailRenderDispatcher.java`**: 轨道渲染的总调度器。管理所有已加载的轨道块 (`RailChunkBase`)，并在每一帧调用 `draw()` 方法。
*   **`rail/BakedRail.java`**:
    *   **职责**: 轨道的预处理类。
    *   **核心逻辑**: 在轨道生成或加载时，根据贝塞尔曲线路径，按照设定的间隔 (`repeatInterval`) 计算出一系列 4x4 变换矩阵 (`Matrix4f`)。这些矩阵代表了轨道模型的每个“切片”的位置和姿态。
*   **`rail/InstancedRailChunk.java`**:
    *   **职责**: 处理具体的渲染提交。
    *   **核心逻辑**: 将 `BakedRail` 计算出的矩阵数据写入 `InstanceBuf`，利用 OpenGL 的 Instancing 技术一次性绘制成百上千个轨道切片。
*   **`rail/MeshBuildingRailChunk.java`**: 备用的渲染实现，如果不支持实例化渲染，则构建静态 Mesh。
*   **`train/RenderTrainD51.java` / `RenderTrainDK3.java`**: 特定型号火车的硬编码渲染器，展示了如何使用该系统渲染复杂的机车模型。
*   **`integration/MtrModelRegistryUtil.java`**: 辅助工具，用于从资源包中加载模型和纹理。

### 2.3 脚本子系统 (`cn.zbx1425.mtrsteamloco.scripting`)

*   **`ScriptHolderBase.java`**:
    *   **职责**: 脚本引擎的宿主。
    *   **核心逻辑**: 初始化 GraalVM Context，配置安全沙盒（限制类加载、禁止 IO），并注入全局变量（如 `Matrices`, `Timing`）。它在一个独立的 `SCRIPT_THREAD` 线程池中执行脚本，避免阻塞主线程。
*   **`ScriptContextManager.java`**: 管理所有活跃的脚本上下文，处理生命周期（创建、销毁）。
*   **`eyecandy/EyeCandyScriptContext.java`**:
    *   **职责**: 专门用于 Eye Candy 对象的脚本上下文。
    *   **核心逻辑**: 充当脚本与 Java 及其它游戏对象之间的桥梁。它维护了 `scriptResult` (读取用) 和 `scriptResultWriting` (写入用) 两个指令队列，实现了双缓冲机制，确保渲染线程安全地消费脚本生成的渲染指令（如 `drawModel`）。
*   **`util/*`**: 包含注入到脚本环境中的各种工具类，如 `TimingUtil` (计时), `GraphicsUtil` (图形辅助)。

### 2.4 数据与资源 (`cn.zbx1425.mtrsteamloco.data`)

*   **`RailModelRegistry.java`**:
    *   **职责**: 加载和管理自定义轨道模型。
    *   **核心逻辑**: 扫描 `mtrsteamloco:rails` 目录下的 JSON 文件，解析模型路径、纹理、脚本等属性，并注册到 `ELEMENTS` Map 中。
*   **`EyeCandyRegistry.java`**:
    *   **职责**: 加载和管理 Eye Candy。
    *   **核心逻辑**: 类似于轨道注册表，解析 `mtrsteamloco:eyecandies` 下的 JSON。它还支持加载物品形式的模型 (`itemModel`)。
*   **`RailModelProperties.java`**: 数据类，存储单个轨道类型的属性（模型、重复间距、Y轴偏移等）。
*   **`EyeCandyProperties.java`**: 数据类，存储单个装饰物的属性（模型、碰撞箱、脚本等）。
*   **`ScriptedCustomTrains.java`**: 似乎用于处理通过脚本定义的自定义列车逻辑。

### 2.5 区块与逻辑 (`cn.zbx1425.mtrsteamloco.block`)

*   **`BlockEyeCandy.java`**:
    *   **职责**: 装饰物方块的定义。
    *   **核心逻辑**:
        *   `getShape()` / `getCollisionShape()`: 根据 `BlockEntity` 中存储的数据动态返回碰撞箱。
        *   `entityInside()`: 实现了检票闸机的逻辑（检测玩家、扣费、开门）。
*   **`BlockEyeCandy.BlockEntityEyeCandy.java`**:
    *   **职责**: 存储 Eye Candy 的实例数据。
    *   **核心逻辑**:
        *   保存变换（平移/旋转/缩放）。
        *   持有 `EyeCandyScriptContext` 引用。
        *   `writeCompoundTag` / `readCompoundTag`: 处理数据的序列化与反序列化（网络同步/存盘）。
*   **`BlockDirectNode.java`**: 一种特殊的节点方块，可能用于构建路径。

### 2.6 物品与工具 (`cn.zbx1425.mtrsteamloco.item`)

*   **`CompoundCreator.java` (复合建造器)**:
    *   **职责**: 强大的轨道修饰工具。
    *   **核心逻辑**:
        *   `onConnect`: 当玩家连接两个节点时触发。
        *   `SliceTask`: 定义了切片任务。算法会遍历两点之间的贝塞尔曲线，计算每个步进点的坐标和旋转，然后放置指定的方块（如生成隧道）。
        *   `RailModifierTask`: 批量修改选定路段的轨道类型或方向。
*   **`RailPathEditor.java`**:
    *   **职责**: 轨道路径编辑器。
    *   **核心逻辑**: 允许玩家点击现有轨道，在服务器端更新其属性（如连接关系），通过 Mixin 暴露的接口修改 MTR 数据。
*   **`RoutePathCreator.java`**:
    *   **职责**: 快速创建行车路线。
    *   **核心逻辑**: 通过连续点击轨道，自动寻找路径并生成 Path Data，简化路线设定流程。
*   **`DisplacementTool.java`**: 位移工具，用于快速传送。

### 2.7 Mixin 集成 (`cn.zbx1425.mtrsteamloco.mixin`)

*   **`RenderTrainsMixin.java`**:
    *   **目标**: `mtr.render.RenderTrains`
    *   **作用**: 拦截 `renderRailStandard`。如果启用了高级渲染配置，则取消原版调用，转而调用 `RailRenderDispatcher` 进行高性能绘制。同时在 `render` 方法的尾部 (`TAIL`) 注入了自定义渲染管线的提交逻辑 (`commit`)。
*   **`RailwayDataRailActionsModuleMixin.java`**:
    *   **目标**: `mtr.data.RailwayDataRailActionsModule`
    *   **作用**: 实现 `RailActionsModuleExtraSupplier` 接口，暴露了受保护的 `railActions` 列表，使得 `CompoundCreator` 可以向系统添加自定义的轨道构建任务。
*   **`RailMixin.java` / `RailAccessor.java`**: 提供对 MTR `Rail` 类内部字段的访问权限。
*   **`ClientCacheAccessor.java`**: 允许访问客户端缓存的数据。

### 2.8 主程序入口 (`cn.zbx1425.mtrsteamloco`)

*   **`Main.java`**: 模组主入口。负责注册方块、物品、数据包接收器 (`PacketHandler`)。
*   **`MainClient.java`**: 客户端入口。负责初始化渲染系统 (`Sowcer`, `RailRenderDispatcher`)，注册客户端事件监听器。
*   **`ClientConfig.java`**: 处理客户端配置文件（渲染距离、优化开关等）。

---

## 3. 总结

MTR-ANTE 通过精心设计的架构，成功在不修改 MTR 源码的前提下，对其图形表现力进行了革命性的升级。其核心价值在于：

1.  **Sowcer 引擎** 解决了 Minecraft 中大量复杂几何体渲染的性能瓶颈。
2.  **GraalVM 脚本系统** 赋予了静态装饰物动态的生命力。
3.  **高级工具集** 极大地提升了铁路建设的效率和精度。

这份代码库展示了高级 Minecraft 模组开发所需的深厚技术功底，包括图形学知识 (OpenGL/Matrix Math)、JVM 内部机制 (Class Loading/Polyglot) 以及对目标系统 (MTR) 的深刻理解。
