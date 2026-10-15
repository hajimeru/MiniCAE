# MiniCAE：Windows + CMake + MSVC 开发方案

> 版本：v2.2 ｜ 更新：2026-10-13  
> 当前目标：Windows x64 原生开发，使用 MSVC 编译器  
> 工具链：VS Code + CMake + Ninja + MSVC + Vulkan SDK + GLFW  
> Linux 开发与远程调试暂缓；Web/WASM 保留后续阶段

本文件由原 Linux 远程编译与 X11 方案调整而来。当前仅准备 Windows 本地工具链；远程同步、SSH、GDB、X11 和 Linux GPU 验证不列入本期任务。原文件已更名，以反映当前用途。

本文是拟实施说明。MiniCAE 与同级 VisCore 已建立 Git 仓库和约定文档，VisCore 已作为子模块接入；尚未建立 CMake 工程。以下配置与命令需在工程实现后验证，不能据此认为本机环境已经配置完成。

## 1. 工具职责与当前范围

| 工具 | 职责 |
|---|---|
| MSVC：cl.exe / link.exe | 编译、链接 Windows C++ 代码 |
| CMake | 配置工程、组织目标、生成构建规则 |
| Conan 2 | 管理第三方依赖，包括纯头文件库，生成 CMake 工具链与包配置 |
| Ninja | 执行 CMake 生成的构建任务 |
| VS Code | 编辑、运行构建任务、调试 |
| Vulkan SDK 与 GPU 驱动 | 开发工具、验证层、Shader 编译与实际运行 |
| GLFW | 窗口、输入、Surface 辅助 |
| Dear ImGui | 按需引入原生控制面板 |
| RenderDoc | GPU 帧捕获与资源分析 |

**Ninja 不是编译器。** 本方案通过 CMake 使用 Ninja 驱动 MSVC；不切换到 MinGW 或 clang-cl。

当前验收平台仅为 Windows x64。跨平台能力通过模块边界保留，Linux 暂标“计划中”，不要求当前安装 Linux 工具、准备远程机器或维护 Linux 配置。

Web 阶段继续使用独立 Emscripten 工具链，不将 MSVC 当成 WASM 编译器。

## 2. 安装与记录工具链

### 2.1 MSVC 与 Windows SDK

安装 Visual Studio 或独立 Build Tools，选择“使用 C++ 的桌面开发”、x64 MSVC 工具和 Windows SDK。可以继续使用已安装且满足项目要求的 VS2022 工具链；不要求安装完整 Visual Studio IDE，也不因新版本发布而强制迁移。

记录实际 MSVC、Windows SDK、CMake、Ninja 和 Vulkan SDK 版本。依赖更新在确认构建和运行结果后再进行。

微软提供开发者命令行初始化编译环境；仅添加 `cl.exe` 所在目录到 PATH 不足以配置头文件、库与工具。[微软官方说明](https://learn.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-170)

### 2.2 CMake、Ninja 与编辑器

安装 CMake 和 Ninja，或使用开发工具附带的版本，避免多套版本混用。VS Code 建议安装：

- CMake Tools：配置、构建、选择目标和调试。
- Microsoft C/C++：MSVC 本地调试；可兼顾初期补全。
- clangd：按需用于补全/诊断；启用时避免与另一套 IntelliSense 重复诊断。

使用 clangd 不会改变 MSVC 编译器选择。优先通过 Ninja 生成 `compile_commands.json`，并让 clangd 指向实际构建目录。

### 2.3 Vulkan 与项目依赖

Vulkan SDK 提供开发工具，运行仍依赖 GPU 驱动及所需能力。检查：

```powershell
vulkaninfo --summary
```

随后由应用验证目标设备、队列、Surface、格式、呈现模式与所需 feature。记录 GPU 和驱动；软件设备结果不当作硬件性能。

第三方库（含 header-only）优先通过 Conan 2 获取固定版本；自研模块通过 Git 子模块固定提交。GLFW、可选 ImGui、Vulkan 开发组件及 Shader 工具的具体包与版本在首次构建设计中确认，GPU 驱动由系统提供。

Conan 独立于 CMake 配置步骤显式运行，通过 CMakeToolchain/CMakeDeps 提供工具链和依赖 target。MiniCAE 集成时统一解析依赖，VisCore 独立开发时使用自己的兼容清单。依赖采用一致的 x64 架构与兼容 MSVC 运行库；记录 lockfile、host/build profiles、Debug/Release 与静态/动态链接策略。

后文 CMake 命令和 Preset 仍是未接入 Conan 的基础模板；工程实现时须接入实际生成的工具链/Presets，不能直接假定它们已包含 Conan 配置。当前尚未安装或生成任何 Conan 工程文件。

## 3. 先打通 MSVC 环境

打开对应安装版本的 **x64 Native Tools Command Prompt**，确认工具：

```bat
where cl
cl
cmake --version
ninja --version
```

`cl` 不带源文件用于查看版本；它可能返回“缺少源文件”的非零状态，不应据此判定编译器未安装。

然后在该命令行中打开项目：

```bat
cd /d E:\Programming\MiniCAE
code .
```

若已有 VS Code 进程没有继承开发者环境，可重启 VS Code 后从该环境启动，或由 CMake Tools 选择并初始化已安装的 x64 MSVC 工具链。普通 PowerShell 默认不保证包含 MSVC 所需环境。

## 4. 构建目录与配置

建议布局：

```text
MiniCAE/
├── CMakeLists.txt
├── CMakePresets.json
├── modules/                     # 独立模块 Git 子模块，待接入
├── src/model/                   # 产品数据与状态
├── src/adapters/                # 模块适配；Vulkan 在 VisCore
├── src/viewer/
├── apps/native/
├── tests/
├── data/
└── build/
    ├── windows-msvc-debug/
    └── windows-msvc-release/
```

核心目标使用 C++17；第三方依赖若要求其他标准，局部处理并记录。Windows/MSVC 专属编译选项只配置到对应目标，不通过全局字符串污染后续 WASM 构建。

同一构建目录不要切换生成器、架构或编译器。Debug/Release 分目录，后续 WASM 也使用单独目录。

### 4.1 直接构建命令

在已初始化的 x64 MSVC 环境中，待 `CMakeLists.txt` 实现后执行：

```powershell
cmake -S . -B build/windows-msvc-debug -G Ninja -DCMAKE_C_COMPILER=cl -DCMAKE_CXX_COMPILER=cl -DCMAKE_BUILD_TYPE=Debug -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
cmake --build build/windows-msvc-debug --parallel
ctest --test-dir build/windows-msvc-debug --output-on-failure
```

确认 CMake 配置输出识别为 MSVC，并检查缓存中的编译器路径；MSVC Debug 构建用于调试，Release 构建用于正式性能测试。

`CMAKE_C_COMPILER` 对纯 C++ 工程可不需要；示例同时指定它，便于后续含 C 代码的依赖采用一致工具链。CTest 需要工程启用并注册测试，不能将“没有测试”视为验收通过。

### 4.2 后续使用 CMakePresets.json 固定配置

下面是配置模板，不是当前已保存的工程文件。它假设开发者环境已初始化；Preset 不负责自动执行 MSVC 环境脚本。

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "windows-msvc-debug",
      "displayName": "Windows x64 MSVC Debug",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/windows-msvc-debug",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "cl",
        "CMAKE_CXX_COMPILER": "cl",
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      }
    },
    {
      "name": "windows-msvc-release",
      "inherits": "windows-msvc-debug",
      "displayName": "Windows x64 MSVC Release",
      "binaryDir": "${sourceDir}/build/windows-msvc-release",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" }
    }
  ],
  "buildPresets": [
    { "name": "windows-msvc-debug", "configurePreset": "windows-msvc-debug" },
    { "name": "windows-msvc-release", "configurePreset": "windows-msvc-release" }
  ],
  "testPresets": [
    {
      "name": "windows-msvc-debug",
      "configurePreset": "windows-msvc-debug",
      "output": { "outputOnFailure": true },
      "execution": { "noTestsAction": "error" }
    }
  ]
}
```

模板需要支持 Presets v3 的 CMake（3.21 或以上）；项目与依赖的最终最低版本另行确定。`x64` 由开发者环境选定，显示名称本身不设置架构；Ninja 不使用 Visual Studio 生成器的 `-A x64` 参数。

模板保存且工程实现后，可使用：

```powershell
cmake --preset windows-msvc-debug
cmake --build --preset windows-msvc-debug
ctest --preset windows-msvc-debug
```

优先 Ninja 是为了简单统一构建与生成编译数据库。Visual Studio 生成器也能使用 MSVC，但它是多配置生成器，要用 `--config Debug` 等参数；`CMAKE_EXPORT_COMPILE_COMMANDS` 不适用于该生成器，不混用这两套说明。[CMake 编译数据库说明](https://cmake.org/cmake/help/latest/variable/CMAKE_EXPORT_COMPILE_COMMANDS.html)

## 5. 本地调试与 GPU 检查

- CMake Tools 选择 `minicae_native` 或核心测试目标，通过 MSVC 调试器启动。
- 如手写 `launch.json`，MSVC 本地调试采用 `cppvsdbg`，不使用 Linux GDB 的 `cppdbg`/pipeTransport 模板。
- `program` 指向真实产物；工作目录与 Shader/样本资源路径在工程中明确，不依赖某个偶然的启动目录。
- CPU 断点依赖 Debug 符号与对应源码；GPU 错误结合验证层和 RenderDoc 排查。
- 所有 Shader 在构建中从 GLSL 编译为 SPIR-V，避免只提交一份可能过期的预编译产物。
- 验证窗口缩放、最小化恢复、重复加载/卸载、关闭以及在途资源释放。

产品规划 `minicae_native` 与产品测试目标；独立渲染库导出 `VisCore::VisCore`。具体产物路径与模块测试以各仓库未来 CMake 实现为准。

## 6. 当前验收清单

- [ ] x64 MSVC、Windows SDK、CMake、Ninja 可用，工具版本有记录。
- [ ] CMake 明确识别为 MSVC，未混入 MinGW 或 clang-cl。
- [ ] 核心可独立构建，实际注册的测试全部通过。
- [ ] Debug 构建可命中本地断点，符号与源码对应。
- [ ] VisCore 独立 GLFW/Vulkan 示例可以显示三角形，实际设备和能力记录齐全。
- [ ] 验证层问题、缩放、退出与资源生命周期检查完成。
- [ ] Debug/Release、依赖运行库和 Shader 构建策略一致。
- [ ] Release 性能记录附硬件、驱动和数据规模。
- [ ] 支持范围写为“Windows x64 已验证”时有实际结果；Linux 写“暂缓/未验证”。

## 7. 暂缓事项与移植边界

Linux 远程源码同步、SSH、GDB、X11、原生图形会话和驱动兼容性全部后移，重新启动该方向时再制定配置，不在当前工程生成相关任务。

保留的边界：独立读取/算法模块与产品数据/流程保持平台边界；文件字节、输入和 Surface 经平台适配传入；VisCore 的 Windows Vulkan Surface 接入留在平台适配层；构建目标区分核心、原生后端和应用。

Web 阶段在 Windows 上安装 Emscripten，建立独立 WASM 构建目录并接入 WebGPU，不要求先完成 Linux 移植。后续 Qt 同样作为应用与窗口适配选择，不进入核心库。

## 8. 官方参考与执行顺序

- [微软 MSVC 命令行环境](https://learn.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-170)
- [CMake Ninja 生成器](https://cmake.org/cmake/help/latest/generator/Ninja.html)
- [CMake Presets](https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html)
- [CMake 编译数据库](https://cmake.org/cmake/help/latest/variable/CMAKE_EXPORT_COMPILE_COMMANDS.html)
- [GLFW Vulkan](https://www.glfw.org/docs/latest/vulkan_guide.html)
- [Vulkan SDK](https://vulkan.lunarg.com/)
- [RenderDoc](https://renderdoc.org/)

当前执行顺序：MSVC/CMake 环境 → 现有模块接入验证 → VisCore GLFW 窗口与 Vulkan 首帧 → CAE 功能 → 性能与原生交付 → Web 最小验证。Linux 不占用当前里程碑。

