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

## 2. 模块详细分析 (按包结构)

### 2.1 主程序包 (`cn.zbx1425.mtrsteamloco`)

*   **`Main.java`**: 模组通用入口（Server/Client）。负责注册方块、物品、粒子效果、声音以及网络数据包接收器。
*   **`MainClient.java`**: 模组客户端入口。负责初始化 Sowcer 渲染系统 (`RailRenderDispatcher`)，注册按键绑定和客户端事件。
*   **`ClientConfig.java`**: 客户端配置管理。处理渲染距离、半透明排序、优化开关等配置项。
*   **`BuildConfig.java`**: (模板生成) 存储模组构建版本、时间等元数据。
*   **`CustomResources.java`**: 处理自定义资源加载逻辑。
*   **`Debug.java`**: 调试辅助工具类。
*   **`KeyMappings.java`**: 定义客户端按键绑定。
*   **`RegistriesWrapper.java`**: 跨平台（Fabric/Forge）注册表包装器，统一处理物品、方块的注册。

### 2.2 方块定义 (`cn.zbx1425.mtrsteamloco.block`)

*   **`BlockEyeCandy.java`**: 核心装饰方块。支持自定义模型、脚本交互、检票闸机逻辑。
*   **`BlockEyeCandy.BlockEntityEyeCandy`**: 存储 Eye Candy 的状态（变换矩阵、脚本上下文、NBT数据）。
*   **`BlockDepartureBell.java`**: 发车铃方块。
*   **`BlockDirectNode.java`**: 路径节点方块，用于构建特定路径。

### 2.3 物品与工具 (`cn.zbx1425.mtrsteamloco.item`)

*   **`CompoundCreator.java`**: 复合建造器。包含 `SliceTask`（切片生成隧道/桥梁）和 `RailModifierTask`（批量修改轨道）逻辑。
*   **`RailPathEditor.java`**: 轨道路径编辑器。用于在服务端修改已生成轨道的连接属性。
*   **`RoutePathCreator.java`**: 路线生成器。通过点击轨道自动生成 Path Data，简化路线设置。
*   **`DisplacementTool.java`**: 位移工具。计算玩家与轨道的相对位置，用于快速移动。
*   **`BlockItemEyeCandy.java`**: Eye Candy 的物品形式，负责渲染物品栏模型。
*   **`BlockItemDirectNode.java`**: Direct Node 的物品形式。

### 2.4 用户界面 (`cn.zbx1425.mtrsteamloco.gui`)

*   **`ScriptDebugOverlay.java`**: 脚本调试覆盖层。显示脚本执行状态和错误信息。
*   **`EyeCandyScreen.java`**: Eye Candy 的配置界面，用于调整模型变换（平移/旋转/缩放）。
*   **`CompoundCreatorScreen.java`**: 复合建造器的配置界面，管理构建任务列表。
*   **`BrushEditRailScreen.java`**: 笔刷编辑轨道界面。
*   **`ConfigScreen.java`**: 模组全局配置界面（使用 Cloth Config）。
*   **`RailPathEditorScreen.java`**: 轨道路径编辑器的 GUI。
*   **`RoutePathCreatorScreen.java`**: 路线生成器的 GUI。
*   **`SelectScreen.java` / `SelectListScreen.java`**: 通用选择列表界面。
*   **`WidgetScrollList.java` / `WidgetSlider.java` / `WidgetLabel.java`**: 自定义 GUI 控件。

### 2.5 网络通信 (`cn.zbx1425.mtrsteamloco.network`)

*   **`PacketScreen.java`**: 处理打开 GUI 的网络包（Server -> Client）。
*   **`PacketUpdateBlockEntity.java`**: 同步 BlockEntity 数据（Client -> Server），如 Eye Candy 的配置更改。
*   **`PacketUpdateRail.java`**: 同步轨道修改操作。
*   **`PacketRoutePathCreator.java`**: 同步路线生成请求。
*   **`PacketReplaceRailNode.java`**: 替换轨道节点请求。
*   **`PacketUpdateHoldingItem.java`**: 同步手持物品更新。
*   **`PacketVersionCheck.java`**: 版本检查与通知。

### 2.6 数据结构 (`cn.zbx1425.mtrsteamloco.data`)

*   **`RailModelRegistry.java`**: 轨道模型注册表。加载 `mtrsteamloco:rails` JSON。
*   **`EyeCandyRegistry.java`**: 装饰物注册表。加载 `mtrsteamloco:eyecandies` JSON。
*   **`RailModelProperties.java`**: 存储轨道模型属性（模型文件、脚本、渲染参数）。
*   **`EyeCandyProperties.java`**: 存储装饰物属性。
*   **`ScriptedCustomTrains.java`**: 脚本化自定义列车数据。
*   **`RailExtraSupplier.java`**: 接口，用于扩展 MTR 的 `Rail` 类。
*   **`RailActionsModuleExtraSupplier.java`**: 接口，用于扩展 MTR 的 `RailActionsModule`。
*   **`ShapeSerializer.java`**: 处理形状数据的序列化。

### 2.7 渲染系统 (`cn.zbx1425.mtrsteamloco.render`)

*   **核心调度**:
    *   **`RailRenderDispatcher.java`**: 轨道渲染调度器，管理 `RailChunk`。
    *   **`RailPicker.java`**: 处理鼠标拾取轨道的逻辑（Raycasting）。
    *   **`RailDistanceRenderer.java`**: 渲染轨道距离提示。
    *   **`RenderUtil.java`**: 渲染工具类（PoseStack 操作等）。
    *   **`ShadersModHandler.java`**: 兼容性处理（如 Iris/OptiFine）。

*   **轨道渲染实现 (`render/rail`)**:
    *   **`BakedRail.java`**: 预计算轨道的变换矩阵。
    *   **`InstancedRailChunk.java`**: 使用 Sowcer 进行实例化渲染的轨道块。
    *   **`MeshBuildingRailChunk.java`**: 降级模式，构建静态 Mesh。

*   **方块渲染 (`render/block`)**:
    *   **`BlockEntityEyeCandyRenderer.java`**: 渲染 Eye Candy（静态模型 + 脚本动态模型）。
    *   **`BlockEntityDirectNodeRenderer.java`**: 渲染节点。

*   **列车渲染 (`render/train`)**:
    *   **`RenderTrainD51.java`**: D51 蒸汽机车渲染器。
    *   **`RenderTrainDK3.java`**: DK3 渲染器。
    *   **`SteamSmokeParticle.java`**: 蒸汽粒子效果。

*   **集成 (`render/integration`)**:
    *   **`MtrModelRegistryUtil.java`**: 资源加载辅助。
    *   **`SowcerModelAgent.java`**: 将模型数据桥接到 Sowcer。
    *   **`DynamicTrainModelLoader.java`**: 动态加载列车模型。

### 2.8 脚本系统 (`cn.zbx1425.mtrsteamloco.scripting`)

*   **引擎核心**:
    *   **`ScriptHolderBase.java`**: GraalVM 宿主，配置 Context 和沙盒。
    *   **`ScriptContextManager.java`**: 上下文生命周期管理。
    *   **`AbstractScriptContext.java`**: 脚本上下文基类。
    *   **`AbstractDrawCalls.java`**: 抽象绘制指令集。

*   **具体实现 (`scripting/eyecandy`)**:
    *   **`EyeCandyScriptContext.java`**: 装饰物专用上下文，实现双缓冲渲染指令队列。

*   **脚本工具 (`scripting/util`)**:
    *   **`TimingUtil.java`**: 提供 `Timing` 对象（帧率、时间）。
    *   **`StateTracker.java`**: 简单的状态追踪器。
    *   **`RateLimit.java`**: 频率限制工具。
    *   **`GlobalRegister.java`**: 全局注册表访问。
    *   **`WrappedEntity.java`**: 实体包装类，安全暴露实体属性给脚本。

### 2.9 Sowcer 渲染引擎 (`cn.zbx1425.sowcer`)

*   **`ContextCapability.java`**: OpenGL 能力检测。
*   **`batch/BatchManager.java`**: 渲染批次优化。
*   **`object/InstanceBuf.java`**: 实例化缓冲区封装。
*   **`object/VertBuf.java`**: 顶点缓冲区封装。
*   **`shader/ShaderManager.java`**: 着色器管理。
*   **`util/OffHeapAllocator.java`**: 堆外内存管理。

### 2.10 Mixin 注入 (`cn.zbx1425.mtrsteamloco.mixin`)

*   **渲染相关**: `RenderTrainsMixin`, `LevelRendererMixin`, `GameRendererMixin`, `JonModelTrainRendererMixin`.
*   **逻辑相关**: `RailwayDataRailActionsModuleMixin`, `RailMixin`, `TrainMixin`, `PathDataAccessor`.
*   **UI/输入**: `KeyboardHandlerMixin`, `ResourcePackCreatorScreenMixin`.

---

## 3. 总结

MTR-ANTE 通过精心设计的架构，成功在不修改 MTR 源码的前提下，对其图形表现力进行了革命性的升级。这份报告覆盖了从底层的 OpenGL 封装 (Sowcer) 到顶层的用户界面 (GUI) 和业务逻辑 (Scripting/Block) 的所有关键组件。
