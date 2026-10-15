# MiniCAE 与独立模块的架构约定

确认日期：2026-10-13。当前工作是建立仓库与约定，不开始实现渲染功能。

## 仓库关系

`E:\Programming\MiniCAE` 为产品仓库，`E:\Programming\VisCore` 为独立库开发仓库。两者各有 Git 历史。将来 MiniCAE 在 `modules/VisCore` 检出 VisCore 指定提交；两个 VisCore 检出目录通过 Git 提交衔接，不自动同步文件。

两个仓库的远端地址：MiniCAE 为 `git@github.com:hajimeru/MiniCAE.git`，VisCore 为 `git@github.com:hajimeru/VisCore.git`。

VisCore 初始文档已发布，产品通过 `modules/VisCore` Git 子模块使用其固定提交；具体版本以产品 gitlink 为准，不使用同级本地路径代替正式远端。VTKFileReader、UnstructureCellPost、SpatialPick 尚未接入，现有其他模块仓库保持不变。

## 目标结构

```text
MiniCAE/
├── AGENTS.md
├── CMakeLists.txt / CMakePresets.json    # 后续实现
├── modules/                            # 后续接入 Git 子模块
│   ├── VisCore/
│   ├── VTKFileReader/
│   ├── UnstructureCellPost/
│   └── SpatialPick/
├── src/
│   ├── model/
│   ├── adapters/
│   └── viewer/
├── apps/native/
├── tests/
├── data/
├── benchmarks/
└── docs/
```

除已有文档外，目录和工程文件按功能创建，当前不生成占位实现。

## 第一个集成闭环

VTKFileReader 读取支持范围内的线性三角形/四面体文件；产品持有数据和原始 ID。UnstructureCellPost 通过视图执行表面提取及后续切面。适配层构造 VisCore 几何/属性，并为 SpatialPick 提供空间数据源。SpatialPick 返回查询结果，产品映射原始实体并请求显示高亮。

显示三角形可能来自体单元表面或算法输出，因此显示 ID 与原始单元 ID 分离。PointData/CellData 转换由产品定义，VisCore 保证其通用属性契约。先小样本，再规模测试。

## 独立模块契约

模块维护公开头文件、命名空间、CMake target、独立测试、安装与消费者示例。模块之间通过公开接口协作；不为共享数据而强迫所有模块采用同一内部容器。产品统一依赖版本并编写适配。

现有 UCP、VTKFileReader 等模块的 CMake 仍需评估作为子目录时的测试/示例开关。本次未运行模块测试，不声明可直接集成或已支持 WASM。

## 依赖管理：Conan 与 Git 子模块

自研模块（包括 header-only 的 UCP）使用 Git 子模块固定提交；第三方库优先使用 Conan 2，不按是否需要编译区分接入方式。产品为一次构建统一解析第三方依赖，再向模块提供 CMake target；子目录不执行独立 Conan 安装。

VisCore 单独开发使用自己的依赖清单；集成到 MiniCAE 时采用产品清单，二者约束保持兼容。开发者在构建前显式执行 Conan，CMake 通过生成的工具链与依赖配置消费结果。Conan 仅管理构建依赖，GPU 驱动不属于包依赖；Vulkan SDK/Shader 工具的具体来源待首个构建设计确定。

复现构建需要固定包版本、recipe revisions/lockfile 与 host/build profiles。lockfile 不替代 MSVC、架构、运行库、Debug/Release 和工具版本记录。未来 WASM 使用独立 profile/构建目录，并单独解析验证依赖图。

极小、稳定且需随源码分发的工具可内置 `third_party/`，记录上游地址、版本或提交、许可证、更新方法和本地修改。需要持续改动的第三方库考虑维护 fork 与自定义 Conan recipe，确有源码开发需求时才选子模块。同一依赖在一次构建中只使用一种来源。

当前只确定依赖管理规则，尚未创建 Conan recipe、profile 或 lockfile，也未执行依赖安装。

## 执行顺序与验收

1. 创建两个本地独立 Git 仓库与 AGENTS.md；核对文档指针与职责一致。
2. 确定真实远端地址与可取得版本，逐个接入子模块；验收干净环境可获取固定提交。
3. 在 VisCore 先完成独立 Vulkan 示例与公开接口；MiniCAE 用 VTK 小样本作为第二消费者验证。
4. 原生 MVP 稳定后接 WebGPU；内部 RHI 根据实际共性演进。

当前阶段的验收是仓库身份、文档与状态检查，不是 GPU 功能测试。未经用户要求不提交或推送。后续功能需单独设计和实现计划。
