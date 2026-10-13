# 从 CAE 渲染工程师到跨平台图形与科学可视化架构师

> **MiniCAE：原生 Vulkan 先行，WebGPU 后续接入**  
> 版本：v2.2 ｜ 更新：2026-10-13  
> 建议投入：每周约 10 小时；24 周约 240 小时  
> 参考周期：2026-10-13 ～ 2027-03-29；按阶段验收推进，可顺延  
> 本期目标：原生 CAE MVP、可复用 C++ 核心、Web 最小验证、性能与架构证据

## 0. 已确定的方向

MiniCAE 是产品集成仓库；VisCore 是同级独立开发的通用渲染与科学可视化库，通过 Git 子模块接入产品。先在 Windows 完成 VisCore Vulkan 示例，再由 MiniCAE 接入 VTK 固定单元数据；后续在 VisCore 中增加 WebGPU 后端。仓库边界与接入流程以 [架构约定](docs/architecture.md) 为准。

**当前开发范围：Windows x64 + CMake + MSVC。** 构建优先使用 Ninja，编译器固定为 MSVC；编辑器使用 Windows 本地 VS Code。Linux 开发、远程同步、GDB、X11 和 Linux GPU 验证暂不安排，不占用当前学习预算。保留平台边界，未来需要时再移植和验证，不宣称已支持 Linux。

本地环境与构建流程见 [Windows CMake + MSVC 开发方案](./Windows%20CMake%20+%20MSVC%20开发方案.md)。

**职业目标：** 在已有 C++、OpenGL、CAE 后处理经验上，深化现代 GPU 编程、计算几何、性能工程和跨平台架构能力，形成可以解释、测试和复用的 C++ 库。

| 项目部分 | 技术选择 | 定位 |
|---|---|---|
| 数据读取与算法 | VTKFileReader、UnstructureCellPost、SpatialPick | 复用独立模块，先验证公开接口与集成 |
| 原生窗口 | GLFW | 起步承载窗口与输入 |
| 通用可视化与原生渲染 | 独立 VisCore：Vulkan、GLSL → SPIR-V | 积累可独立消费的库与底层 GPU 能力 |
| 原生控制面板 | Dear ImGui，按需引入 | 字段、切面、统计等工具界面 |
| Web 核心 | Emscripten / WASM | 复用 C++ 数据与算法 |
| Web 渲染 | C++ WebGPU API + Emdawnwebgpu、WGSL | 第二个渲染后端 |
| Web 页面 | TypeScript、Vite、HTML/CSS | 界面、文件选择、启动与异步连接 |
| Qt | 后续可选 | 复杂桌面工作流出现后评估 |
| OSG、Three.js | 不作为本项目依赖 | 既有经验或对照资料 |
| bgfx、Dawn Native、DX12、Metal | 按问题研究 | 不增加本期生产后端 |

C++17 为项目核心基线；所锁定的第三方依赖或构建工具若要求更新标准，局部调整并记录原因。本表是设计选择，不代表依赖已经安装、版本已经锁定或平台已经验证。

**重要取舍：** Vulkan 后端不能直接编译成 WASM 在浏览器运行。选择两套后端，是为了同时获得原生底层经验与 Web 能力；渲染实现的复用程度低于两端都直接使用 WebGPU 的路线。

## 1. 范围与优先级

### 1.1 原生 MVP

- 使用 VTKFileReader 支持编码的小型线性三角形/四面体 VTK 文件；通过 UCP 提取体网格表面。
- 旋转、平移、缩放、相机复位与稳定的单元 ID。
- 节点标量场、单元中心常量场、色带、图例和字段切换。
- CPU 射线拾取、AABB/BVH、四面体平面切面。
- 错误反馈、重复加载/卸载、窗口缩放和资源释放。
- 固定样本下的正确性测试、性能记录和 Windows/MSVC 原生验证。

先用 CPU 拾取和切面打通产品；GPU 拾取与 Compute 是有明确价值时的扩展，不将算法全部 GPU 化。

### 1.2 本期 Web 最小验证

同一小型数据集可以显示、交互、切换标量场，并验证共享 C++ 查询结果。目标是证明架构可扩展；完整加载流程、大规模优化和原生功能全面对齐可放到后续阶段。

### 1.3 暂缓内容

全求解器格式、高阶/任意多面体全覆盖、完整 CAD 编辑、商业级 GUI、通用场景图、全功能 RHI、光追、Mesh Shader、多队列优化、自动 Shader 跨译、Qt 集成、bgfx 集成。

**时间不足时的顺序：** 正确性 → 原生可用 → 性能证据 → Web 小样本验证 → 扩大双端功能。不得把未完成的后端写成已支持。

## 2. 架构：产品集成与独立可视化库

```text
VTK 文件 → VTKFileReader → MiniCAE 产品数据与原始 ID
                                ↓
                    UCP 算法 / SpatialPick 查询
                                ↓
                       产品适配与查看器流程
                                ↓
                     VisCore 通用可视化接口
                                ↓
                      内部图形抽象与后端
                         ↙            ↘
                      Vulkan        WebGPU（后续）
```

具体仓库布局、模块职责和 Git 子模块流程见 [架构约定](docs/architecture.md)，执行规则见 [AGENTS.md](AGENTS.md)。VisCore 规则与设计独立维护在其仓库。

### 2.1 数据边界

读取库保留数据格式语义；MiniCAE 持有产品数据、字段和原始 ID，并通过适配层向算法、拾取和渲染模块提供输入。避免重写已有 MeshCore、BVH 与切面生产实现；需要参考算法用于学习/对照时与生产模块区分。

VisCore 接收通用几何、属性、相机和不透明 ID，不依赖 VTK、UCP 或 SpatialPick。通用相机状态/数学归 VisCore；输入连接和 CAE 操作归产品。

### 2.2 可视化契约与内部 RHI

公开接口先覆盖几何上传/替换、标量属性、色带、视图、可见性和高亮。明确数据所有权、借用有效期、更新版本、能力查询与错误。产品拾取/异步请求保留独立职责，不强迫 VisCore 实现 SpatialPick 的算法接口。

Vulkan 后端管理 GPU 资源、命令、同步和销毁。呈现目标由平台适配接入，应用拥有事件循环和帧调度。先形成具体 Vulkan 实现，WebGPU 后端出现时再提炼稳定 RHI；初期不公开完整底层图形接口。

### 2.3 一致性要求

- 区分节点场插值与单元场常量；产品适配属性关联，库保证其插值契约。MVP 可拆分顶点表达单元常量，并记录内存代价。
- 显示图元 ID 与原始体单元 ID 分离，算法输出保留来源映射。
- 定义坐标、矩阵、深度、面朝向、屏幕坐标和 GPU 数据布局；不假定 C++ 与 Shader 结构布局相同。
- 首期 Vulkan GLSL/SPIR-V 与后续 WGSL 分别维护，以相同公式和样本验证。
- 公共功能按后端能力实现，专有增强显式查询；设备错误、异步读回和在途释放由后端处理。

## 3. 仓库与构建

MiniCAE 规划 `modules/` 子模块、`src/model`、`src/adapters`、`src/viewer` 与 `apps/native`；VisCore 规划公开头文件、可视化实现、Vulkan 后端、独立示例和安装包。完整结构以架构约定为准，按功能创建目录。

产品正式构建使用 `modules/VisCore`，同级 `E:\Programming\VisCore` 用于独立开发。VisCore 已发布初始文档并通过子模块接入，具体版本以产品 gitlink 为准；其余算法/读取模块待接入。

公开渲染 target 规划为 `VisCore::VisCore`；产品目标规划为 `minicae_native` 与产品测试。现有模块有各自 target 与构建选项，集成前验证。两仓库当前没有 CMake 工程，相关命令仍是模板。

## 4. 里程碑与时间预算

以下 24 周是学习预算和推进顺序，双端完整产品不作为 240 小时内的硬承诺。每周约 6 小时编码、3 小时资料阅读、1 小时测试复盘。日期可整体顺延。

| 阶段 | 时间 | 交付 | 进入下一阶段的条件 |
|---|---|---|---|
| M1：原生基础 | W1—W4 | Vulkan 网格、相机、资源基础 | 上传、缩放、释放正确，验证层问题已处理 |
| M2：原生 CAE MVP | W5—W8 | 云图、字段切换、CPU/BVH 拾取 | 场语义和稳定 ID 正确，有首轮基准 |
| M3：体网格与底层深化 | W9—W12 | CPU 切面、Compute 小实验 | 退化测试、同步验证和数值参考通过 |
| M4：性能与原生交付 | W13—W16 | 性能报告、Windows/MSVC 原生验证 | 可重复构建、明确规模边界、可演示 |
| M5：Web 最小验证 | W17—W20 | WASM 核心、WebGPU 小样本渲染 | 共享核心运行、第二后端验证契约 |
| M6：收敛与展示 | W21—W24 | 一致性报告、文档、发布材料 | 如实声明功能、平台与未完成范围 |

若 W16 原生 MVP 未通过验收，后移 Web 阶段。允许最终交付“原生 v1.0 + Web 原型”，不以赶日期为由牺牲正确性。

## 5. 24 周详细任务

每周至少交付一个可运行结果或可核对记录，并在 `docs/weekly/Wxx.md` 记录问题、依据、剩余风险和下一步。

| 周 | 任务 | 验收与交付 |
|---|---|---|
| W01 | Windows x64、MSVC、CMake/Ninja、模块接入检查、VisCore GLFW/Vulkan 环境 | 空窗口、设备/队列能力日志、最小核心测试；本地构建与调试记录 |
| W02 | Instance、Device、Surface、Swapchain、验证层、三角形 | 可显示、缩放、最小化恢复和退出；处理相关验证错误 |
| W03 | Vertex/Index Buffer、staging 上传、GLSL/SPIR-V、深度测试 | 自建索引网格；解释上传、绘制与完成之间的依赖 |
| W04 | 相机控制、基础资源封装、帧资源与销毁 | 旋转/缩放/平移；持续缩放、重载无崩溃；v0.1 |
| W05 | VTKFileReader 数据与 UCP 视图适配、产品字段和 ID 映射 | 场关联和长度验证、稳定 ID；验证模块消费，不重写已有网格库 |
| W06 | LUT、节点插值、单元常量、无效值、图例；按需 ImGui | 小样本值/颜色映射验证；恒定场、NaN/Inf 等行为明确 |
| W07 | SpatialPick 适配、原始 ID 映射与拾取验证 | 对照暴力查询；退化、边界、随机场景测试 |
| W08 | 字段切换、高亮、查看器状态与小型渲染契约 | 10 万三角形基础交互；首轮加载/拾取/内存记录；v0.2 |
| W09 | VTK 四面体数据适配与 UCP 表面提取 | 单/相邻四面体，内部面去除、方向与字段关联正确 |
| W10 | 接入 UCP 平面切面与字段输出 | 平面穿顶点/边/面、重复点、容差与相邻单元测试 |
| W11 | Vulkan Compute 小实验：标量 min/max 归约 | 与 CPU 参考对比；定义空输入、无效值和误差 |
| W12 | 上传→Compute→读回同步；帧资源、Fence/Semaphore/Barrier | 相关同步验证问题已处理；分离上传/执行/读回成本；v0.3 |
| W13 | 最简版本化网格格式、后台解析、交付主线程 | 文件截断/尺寸溢出/非法索引处理；加载中仍可操作 |
| W14 | 分块、内存预算、延迟释放与取消 | 重复加载/卸载/字段切换；内存记录，无持续异常增长 |
| W15 | CPU/GPU Profile，优化一个实测瓶颈 | RenderDoc 或适用工具证据；优化前后和正确性回归 |
| W16 | Windows/MSVC 原生验证、规模测试、依赖固定 | 10万/100万及可运行更大样本；失败边界、构建说明；原生 MVP |
| W17 | Emscripten 验证读取/算法模块的可用子集，页面调用与文件字节边界 | 同一算法在原生/WASM 测试；复制、视图、内存增长说明 |
| W18 | Emdawnwebgpu、Canvas、异步初始化和帧调度 | C++ WebGPU 三角形/小网格；不可用时明确提示 |
| W19 | WebGPU 云图、相机、字段切换、WGSL 布局 | 同一数据的节点/单元场语义；验证 VisCore 可视化契约 |
| W20 | Web 端验证 SpatialPick 可用子集、双端数值与显示验证 | 几何、ID、场值对照；Web 原型、架构差异记录 |
| W21 | 收敛接口、错误恢复、加载/取消/释放 | 过期请求不覆盖新模型；记录设备丢失与重建策略 |
| W22 | Windows/MSVC 自动构建与测试、依赖/Shader 工具版本记录 | Windows 原生构建与核心测试；Web 阶段完成后加入 WASM 核心测试和 Web 构建 |
| W23 | 原生演示、可选在线 Web 原型、优化案例 | 新环境步骤、合法公开样本、性能环境和支持矩阵 |
| W24 | 项目复盘、接口示例、发布标签与作品说明 | 可讲清设计/正确性/性能；明确原生与 Web 各自完成度 |

现有读取/算法库本次未运行测试，也未验证 WASM；接入、构建选项补强与移植可能增加预算。W1—W4 的 Vulkan 初始化允许超出预算；出现阻塞时缩小当前样本和功能。不要用大而全的示例框架隐藏需要掌握的资源和同步机制，也不要求从零实现所有辅助工具。

## 6. 需要真正掌握的知识

### 6.1 Vulkan 原生后端

理解 Instance/PhysicalDevice/Device、队列与呈现能力、内存类型与 staging、Buffer/Image、Descriptor、Pipeline、Command、Swapchain、帧资源与销毁。

同步是显式责任：区分 GPU 内存可见性、队列执行依赖及 CPU 等待；通过验证层和最小例子检查。队列等待或设备等待可用于简单初始化/关闭，但不作为每帧资源管理的长期设计。[Vulkan 同步指南](https://docs.vulkan.org/guide/latest/synchronization.html)

项目建议先验证 Vulkan 1.3 开发基线及实际所需特性；若目标机器不满足，记录并决定降低基线或排除支持，不提前承诺旧系统/所有国产 GPU。API 版本不能代替 feature 与 Surface 支持检查。

### 6.2 C++ 核心与几何

明确索引宽度、节点/单元 ID、结构化数组布局、数据所有权；测试非法网格、退化几何与容差。BVH 对照暴力算法；切面先限定四面体，CPU 参考实现保留用于后端验证。

### 6.3 WebGPU 与 WASM

WebGPU 是另一套后端接口，不是浏览器 Vulkan。WebGPU 资源用途、BindGroup、管线、WGSL、异步读回和浏览器能力需要单独理解。

C++ WebGPU 后端通过 Emdawnwebgpu 接入浏览器 API；Dawn Native 可用于单独实验，但不替代本项目 Vulkan 后端。[官方 C++ 双端示例](https://developer.chrome.com/docs/web-platform/webgpu/build-app)

共享 C++ 核心经 Emscripten 编译；JS/WASM 的大数组边界要明确复制、内存视图和增长失效。浏览器主线程不能依赖原生阻塞循环。初期单线程 WASM；需要多线程时再处理 Worker、部署隔离条件和目标兼容性。

### 6.4 架构与框架研究

把“可复用库”落实为稳定接口、数据语义、测试、构建和示例。bgfx 的句柄、能力查询与后端组织可参考，但不是生产依赖。GLFW 提供 Vulkan 窗口/Surface 辅助，不代替 Device、Swapchain 或同步管理。[GLFW Vulkan 指南](https://www.glfw.org/docs/latest/vulkan_guide.html)

## 7. 性能与一致性验证

### 7.1 测量链路

`文件读取 → 解析 → 拓扑/索引 → 几何算法 → GPU 上传 → 首帧 → 稳定交互 → 释放`

分别记录 CPU 时间、GPU 时间、传输/读回时间、规模、内存与环境。GPU 计时能力不可用时注明，不能把 CPU 提交耗时当 GPU 执行耗时。

| 样本 | 用途 |
|---|---|
| 几个至数千单元 | 语义、退化、容差和 ID 正确性 |
| 10 万三角形 | 原生基础交互与首轮基准 |
| 100 万三角形 | 加载、上传、查询与资源压力 |
| 尝试 1000 万三角形 | 可选压力测试；记录成功子集或失败原因 |

三角形数、节点数、体单元数分别统计。没有明确硬件、数据和算法定义，不承诺固定 FPS 或切面耗时。跨原生/Web 测量要列出实际 GPU 与环境，结果差异不能简单归因于 API。

### 7.2 双端验证

共用输入数据和 CPU 数值参考。原生/WASM 算法比较 ID、拓扑和数值，浮点结果按定义的容差比较。渲染比较场语义、色带范围、方向、深度和选择结果；截图比较需固定相机/分辨率并允许光栅化差异。

CPU 参考→Vulkan Compute 的测试本期完成；WebGPU Compute 的同任务对照是后续扩展，不作为 W20 Web 最小验证硬门槛。

### 7.3 基准报告模板

```markdown
# Case：名称
- 数据：节点/单元/三角形/字段数，真实字节与来源
- 环境：CPU/GPU/驱动/系统/浏览器/构建模式
- 操作：上传、BVH 建立、查询、切面或某个渲染 Pass
- 对照版本：优化前/后，提交或版本标识
- 方法：预热、重复次数、计时方式、中位数；样本足够时 P95
- 正确性：参考结果、容差、边界用例
- 结果：时间、内存、改善及适用范围
- 限制：不可用能力、失败规模、兼容性和维护代价
```

## 8. 工程验收

- [ ] 读取/算法模块可独立消费；VisCore 有独立示例和安装消费者验证。
- [ ] 原生端网格、云图、字段切换、拾取、四面体切面可演示。
- [ ] 节点与单元场语义、无效值处理、稳定 ID 有测试依据。
- [ ] 资源上传、缩放、重复加载/卸载和关闭路径正确。
- [ ] Vulkan 验证问题被修复或逐项记录原因，不以“无日志”等同绝对正确。
- [ ] Windows x64/MSVC 的构建、调试与实际 GPU 渲染分别记录；Linux 标记为未来计划，不作为本期验收项。
- [ ] 原生 v1.0 与 Web 原型分别声明功能和支持范围。
- [ ] WASM 核心和 WebGPU 小样本验证实际通过后才标记支持。
- [ ] 锁定依赖、构建工具和 Shader 编译器；有新环境构建说明。
- [ ] README、架构、接口生命周期、性能案例与演示齐全。
- [ ] 公开代码/数据不含公司内部资源。

CPU 核心测试、构建检查可以自动化；GPU 测试需要明确的设备与运行环境。CI 编译通过不能证明 GPU 功能已验证。

## 9. 执行节奏与防跑偏

每周复盘：目标、实际交付、最重要的理解、失败及解决证据、测试/性能、下一周唯一优先事项。

延期先减少 UI 细节、格式和单元类型，再后移 Web 功能；保留核心语义和算法验证。真正的阻塞缩成最小例子，核对规范与工具版本。AI 适合辅助构建、定位和整理，资源生命周期、同步、布局、性能结论要自己验证并能解释。

不要为“积累库”提前建设无使用者的通用引擎。每个模块应回答：服务哪个功能、输入输出是什么、谁拥有数据、怎样证明正确、为何值得独立。

## 10. 前三次任务

1. **环境与核心：** 确认 Windows x64/MSVC、CMake/Ninja、Vulkan 驱动与工具可用；验证独立模块构建和 VisCore 示例；记录设备能力。所有构建与调试先在 Windows 本地完成。
2. **第一幅画面：** GLFW 创建无 OpenGL 上下文的窗口；Vulkan 完成三角形与验证日志；能缩放和正常退出。环境未知时不承诺两小时完成。
3. **从三角形到数据：** 显示自己生成的索引网格，加入相机；梳理数据上传与资源销毁，提交截图和问题记录。

首个可运行 Vulkan 网格出现之前，暂不接 Qt、bgfx、Web 前端或第二后端。

## 11. 官方资料与后续方向

- [Khronos Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/)
- [Vulkan Guide](https://docs.vulkan.org/guide/latest/)
- [GLFW Vulkan Guide](https://www.glfw.org/docs/latest/vulkan_guide.html)
- [Dear ImGui](https://github.com/ocornut/imgui)
- [Vulkan Memory Allocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)：按需使用；仍需理解分配与同步
- [Emscripten](https://emscripten.org/docs/)
- [Emdawnwebgpu](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/emdawnwebgpu/README.md)
- [C++ WebGPU 跨平台应用](https://developer.chrome.com/docs/web-platform/webgpu/build-app)
- [WebGPU Specification](https://www.w3.org/TR/webgpu/)
- [WGSL Specification](https://www.w3.org/TR/WGSL/)
- [TypeScript](https://www.typescriptlang.org/docs/)、[Vite](https://vite.dev/guide/)、[CMake](https://cmake.org/cmake/help/latest/)
- [bgfx](https://bkaradzic.github.io/bgfx/overview.html)：架构对照，不作为本期依赖

后续按真实目标选择：原生引擎方向深入 Vulkan 显存/同步/多线程与 GPU Profile；Web CAD 方向扩大 WebGPU/WASM、分块加载与产品界面；CAE 核心方向深入多面体、等值面和 out-of-core。就业价值是选择依据之一，不把少量岗位样本当成全市场数量或未来保证。

**成果标准：能独立设计、实现、验证和解释 CAE 数据、几何、GPU 资源与跨端边界，而不只展示使用了哪些 API。**

