# pyFEM 帮助文档（完整版）

## 说明

- 本文件由《pyFEM帮助文档正文（第一批）》到《pyFEM帮助文档正文（第六批）》合并整理而成。
- 文档内容严格基于 `project_code_bundle.txt` 中可确认的实现与测试行为编写，保持保守口径，不把占位能力、预留入口或未接入正式主线的对象写成已支持功能。
- 当前版本的重点是形成一份连续可读、章节统一、术语一致的完整帮助文档，便于后续继续润色、补图或转换为发布格式。

## 目录

1. 第 1 章 软件概览
2. 第 2 章 安装与启动
3. 第 3 章 快速开始
4. 第 4 章 定义层与编译层
5. 第 5 章 输入文件参考
6. 第 6 章 GUI 工作台
7. 第 7 章 Python API / Facade / Probe
8. 第 8 章 建模对象说明
9. 第 9 章 Procedures 层
10. 第 10 章 Kernel 层
11. 第 11 章 Solver 层
12. 第 12 章 结果层
13. 第 13 章 作业、快照与派生算例
14. 第 14 章 插件与扩展说明
15. 第 15 章 常见错误与排查
16. 第 16 章 开发与维护说明
17. 第 17 章 术语表

---

## 第 1 章 软件概览

本章是整套帮助文档的总入口。它首先回答的不是“某个对象有哪些字段”，而是“pyFEM 当前采用什么正式分层、每一层为什么存在、层与层之间怎么衔接”。

当前文档不再继续沿用“定义层 / 编译层 / 运行时层 / 结果层 / GUI 消费层”这一过于压缩的五层简写。原因很简单：源码中的 `kernel`、`procedures`、`solver` 已经具有独立职责，如果仍然把它们合并成“运行时层”大桶，就会同时看不清对象边界、执行边界和产品壳层边界。

### 1. 功能概述

当前版本能够由源码直接确认的正式主线如下：

1. 通过 INP、GUI 或 Python API 形成定义层对象。
2. 通过 `Compiler` 将定义层编译为 `CompiledModel`。
3. 在 `CompiledModel` 中装配 `kernel runtime` 与 `ProcedureRuntime`。
4. 由 procedures 层驱动分析步执行，并通过 solver 层完成离散系统求解。
5. 将结果写入正式结果对象层级，再由 facade、probe、VTK 或 GUI 消费。

这条主线里的关键变化是：`kernel`、`procedures`、`solver` 从本章开始被视为三个不同的正式层，而不是同一个“运行时层”的内部子项。

### 2. 适用范围

当前版本已经确认的正式支持范围如下。

| 能力域 | 当前正式支持 |
| --- | --- |
| 分析类型 | `static` / `static_linear`、`static_nonlinear`、`modal`、`implicit_dynamic` |
| 单元类型 | `B21`、`CPS4`、`C3D8` |
| 材料模型 | `linear_elastic`、`j2_plasticity` |
| 截面类型 | `solid`、`plane_stress`、`plane_strain`、`beam` |
| 边界条件 | `displacement`，目标为 `node` 或 `node_set` |
| 载荷 | 节点载荷；`C3D8 + surface + pressure` 的分布载荷主线 |
| 结果后端 | `json` reader/writer、`memory` writer |
| 结果消费 | `ResultsFacade`、probe、CSV probe 导出、VTK 导出 |
| GUI 主线 | 打开模型、编辑正式对象、提交作业、打开结果、probe、导出 VTK |

当前版本不应写成正式支持的内容包括：

- 显式动力学、热分析、屈曲分析、接触分析。
- 壳单元、高阶单元、三角形或四面体单元。
- `follower_pressure`、`traction`、`nlgeom=True` 下的 distributed load 主线。
- 完整接触/耦合/tie 工作流、网格生成、优化流程、截图导出、正式 Python 控制台接入。

### 3. 正式分层

#### 3.1 为什么需要新的分层口径

新的分层口径不是为了增加术语，而是为了稳定回答以下问题：

1. “定义对象”与“运行时对象”到底在哪里切开。
2. `CompiledModel` 到底是编译产物，还是求解器对象。
3. `kernel` 为什么不等于 `procedures`。
4. `solver` 为什么不应再被 API、GUI、job 壳层直接叙述。

下表先给出当前帮助文档采用的正式层次地图。

| 层次 | 代表对象 | 这一层回答的问题 | 当前正式边界 |
| --- | --- | --- | --- |
| 定义层 | `ModelDB`、`Part`、`Mesh`、`Assembly`、`StepDef`、`JobDef` | “要算什么” | 保存模型定义、命名对象、工况、步骤与作业顺序 |
| 编译层 | `CompilationScope`、`Compiler`、`DofManager`、`CompiledModel` | “如何把定义翻译成可执行对象图” | 解析作用域、建立自由度、绑定 provider，并产出编译结果 |
| kernel 层 | `MaterialRuntime`、`SectionRuntime`、`ElementRuntime`、`ConstraintRuntime`、`InteractionRuntime` | “局部物理对象是什么” | 定义材料、截面、单元、约束和相互作用的运行时接口 |
| procedures 层 | `ProcedureRuntime` 及各步骤过程实现 | “这一分析步如何推进” | 组织步骤执行、状态继承、结果写出和对 solver 的调用 |
| solver 层 | `Assembler`、`DiscreteProblem`、`LinearAlgebraBackend`、`RuntimeState`、`StateManager` | “全局离散系统如何形成并求解” | 处理装配、约束缩并、线性代数求解和状态管理 |
| 结果层 | `ResultsWriter`、`ResultsDatabase`、`ResultStep`、`ResultFrame`、`ResultsFacade` | “结果如何组织、写出、读取和查询” | 保存正式结果层级，并提供 reader/query/probe 入口 |
| 产品壳层 / 消费层 | `PyFEMSession`、`JobManager`、GUI 工作台、VTK exporter | “用户如何消费主线能力” | 作为入口壳层调度编译、作业和结果消费，不直接定义 solver 内部合同 |

需要特别记住两点：

1. `CompiledModel` 归属于编译层，它持有运行时对象图，但本身不是 kernel、procedures 或 solver 的具体实现类。
2. `ResultsFacade` 归属于结果消费侧，它是结果浏览入口，不是结果数据库本体。

#### 3.2 最容易混淆的边界

下表用于澄清最容易被混写的对象或层边界。

| 对象或边界对 | 所属层 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- | --- |
| `ModelDB` | 定义层 | 问题定义唯一来源 | 可直接运行的求解对象 | 经常被误写成“运行时模型” |
| `CompiledModel` | 编译层 | 编译产物与运行时对象图持有者 | solver 本体 | 因其持有 runtime 字段，常被误归到“运行时层” |
| `StepDef` | 定义层 | 分析步定义对象 | 执行步骤的过程对象 | 它定义“这一步用什么”，不负责执行 |
| `ProcedureRuntime` | procedures 层 | 分析步执行对象 | 步骤定义对象 | 它消费 `StepDef`，而不是替代 `StepDef` |
| `ElementRuntime` | kernel 层 | 单元局部运行时接口 | 全局装配器 | 它只提供局部贡献，不组织整体求解 |
| `Assembler` | solver 层 | 全局装配与系统形成对象 | 单元运行时 | 它消费 kernel 贡献，不定义材料或单元物理 |
| `ResultsDatabase` | 结果层 | 正式结果对象树 | GUI 浏览器 | 它是结果数据本体 |
| `ResultsFacade` | 结果层消费侧 | 对结果数据库的查询门面 | 结果数据库本体 | 经常被误写成“结果层唯一对象” |
| `JobManager` / `PyFEMSession` | 产品壳层 | 用户入口或作业入口 | solver 内部对象 | 它们调度主线，不承担离散求解职责 |

### 4. 全局主线

#### 4.1 从定义到结果的正式主线

当前版本应按下面的顺序理解正式主线，而不是跳过 `kernel` 或 `solver`：

```text
INP / GUI / Python API
    -> ModelDB
    -> CompilationScope / Compiler / DofManager
    -> CompiledModel
    -> kernel runtimes
    -> ProcedureRuntime
    -> Assembler / DiscreteProblem / RuntimeState
    -> ResultsWriter / ResultsDatabase
    -> ResultsFacade / Query / Probe / VTK / GUI
```

这条链路分别回答的是：

1. 模型如何被定义。
2. 定义对象如何被解析为 canonical scope 与编译结果。
3. 编译后保存了哪些正式运行时对象。
4. 局部材料、截面、单元、约束由谁表示。
5. 分析步由谁组织和推进。
6. 全局离散系统由谁形成和求解。
7. 结果由谁保存、由谁消费。

#### 4.2 一条最小对象链

如果只沿对象主线看一次最小流程，可以压缩为：

```text
SectionDef / BoundaryDef / StepDef
    -> Compiler.compile(ModelDB)
    -> CompiledModel
    -> ProcedureRuntime.run(...)
    -> ResultsDatabase
    -> ResultsFacade
```

这条对象链的重点不是“每一步有哪些字段”，而是明确：定义对象、编译对象、runtime 对象、solver 对象、结果对象和消费对象处在不同层。

### 5. 使用方式

当前最主要的三种使用方式如下。

| 使用方式 | 适用场景 | 当前入口 | 主要触达层 |
| --- | --- | --- | --- |
| INP 脚本主线 | 跑通最小算例、批量计算、验证输入文件 | `run_case.py` | 定义层 -> 编译层 -> procedures / solver -> 结果层 |
| GUI 主线 | 打开模型、编辑正式对象、提交作业、浏览结果 | `run_gui.py` 或 `pyfem-gui` | 产品壳层 / 消费层 |
| Python API 主线 | 二次开发、脚本集成、自动化结果消费 | `pyfem.api.session.PyFEMSession` | 产品壳层，向下调度定义、编译和结果消费 |

### 6. 结果与输出

当前版本的正式结果主文件通常是 `.results.json`。其中会保存：

- `ResultsSession` 级元数据。
- `ResultStep` 级步骤信息。
- `ResultFrame` 与 `ResultField`。
- `ResultHistorySeries`。
- `ResultSummary`。

如果启用 VTK 导出，还会得到 `.vtk` 文件，适合做几何和结果场可视化。

### 7. 限制与注意事项

1. 当前代码中产品命名尚未完全统一，文档统一使用“pyFEM”作总称。
2. `pyproject.toml` 未完整声明核心运行依赖，不能把“读到的依赖文件”直接等同于“官方安装说明”。
3. 当前没有统一的单位制约定，文档不能擅自固定为 SI 或 N-mm-MPa。
4. GUI 中有大量可见但未完成的预留入口，不能把这些入口写成已经形成正式工作流的功能。
5. 多步结果结构已经存在，但标准一键作业入口一次主要执行一个解析步骤；两者不能混为一谈。

### 8. 常见错误与排查

| 现象 | 常见原因 | 处理方法 |
| --- | --- | --- |
| 把 `kernel`、`procedures`、`solver` 当成同一层 | 仍沿用旧的“运行时层”简写 | 阅读后续章节时优先按三层分工理解 |
| 把 `CompiledModel` 当成求解器本体 | 只看到它持有 runtime 字段 | 记住它是编译层产物和对象图容器 |
| 把 `StepDef` 当成执行对象 | 只看到了步骤参数 | 将 `StepDef` 与 `ProcedureRuntime` 分开理解 |
| 把 `ResultsFacade` 当成结果数据库 | 只从 API 入口使用结果 | 区分“结果本体”和“结果门面” |
| 把 GUI 可见按钮当成正式支持能力 | 以界面可见性代替模块边界判断 | 以源码主线和正式章节说明为准 |

### 9. 本章主线图

```text
定义层
    -> 编译层
    -> kernel 层
    -> procedures 层
    -> solver 层
    -> 结果层
    -> 产品壳层 / 消费层
```

### 10. 相关主题

- 第 2 章 安装与启动
- 第 3 章 快速开始
- 第 4 章 定义层与编译层
- 第 12 章 结果层

---

## 第 2 章 安装与启动

本章位于“概览之后、正式求解之前”的位置，用于说明当前版本需要什么运行环境、有哪些入口、怎样完成最小启动验证。

### 1. 功能概述

本章只回答三个实际问题：

1. 运行 pyFEM 需要准备什么环境。
2. 当前有哪些正式入口可以启动。
3. 如何在不依赖正文后续章节的情况下先验证环境可用。

### 2. 适用范围

本章当前只覆盖“源码目录直接运行”的启动方式。

不在本章正式范围内的内容包括：

- 未确认的 `pip install` 发布流程。
- 未确认的二进制安装包或独立桌面发行方式。
- 未确认的平台支持矩阵。

### 3. 环境要求

#### 3.1 Python 版本

当前 `pyproject.toml` 明确要求 Python `>=3.11`。

#### 3.2 代码中可确认的运行依赖

| 依赖类别 | 当前可确认情况 |
| --- | --- |
| Python | 必需，版本 `>=3.11` |
| `numpy` | 求解后端代码直接使用 |
| `scipy` | 线性方程与特征值求解代码直接使用 |
| `PySide6` | GUI 可选依赖 |
| `pyvista` | GUI 3D 预览可选依赖 |
| `pyvistaqt` | GUI 3D 预览可选依赖 |

说明：

- 当前 `pyproject.toml` 只明确列出了 GUI 可选依赖。
- 由于求解主线代码直接导入 `numpy` 与 `scipy`，实际运行前应确认这两个数值库已经存在。

### 4. 启动入口

#### 4.1 GUI 启动

当前项目中可以确认两种 GUI 入口形式，但对“源码目录直接运行”来说，应优先使用 `run_gui.py`：

```bash
python run_gui.py
```

`pyfem-gui` 也已经在 `pyproject.toml` 中声明为脚本入口，但它是否可以在当前环境中直接调用，取决于实际安装方式是否已完成。因此，本章不把它当作默认必然可用的启动命令。

#### 4.2 INP 求解脚本启动

最直接的脚本入口是：

```bash
python run_case.py
```

`run_case.py` 支持通过环境变量覆盖输入、结果与导出路径，不必修改脚本主体。

#### 4.3 Python API 启动

如果采用 Python API，可直接使用 `PyFEMSession`：

```python
from pyfem.api import PyFEMSession

session = PyFEMSession()
report = session.run_input_file("beam.inp")
results = session.open_results(report.results_path)
print(report.step_name)
print(results.list_steps())
```

说明：

- 这条路径适合脚本集成或二次开发。
- 如果不是从项目源码根目录运行，需要先保证 `pyfem` 包可被 Python 正常导入。

### 5. `run_case.py` 的常用环境变量

| 环境变量 | 作用 | 未设置时的行为 |
| --- | --- | --- |
| `PYFEM_INP_PATH` | 输入 INP 文件路径 | 使用脚本顶部的 `INP_PATH`，默认是 `Job-1.inp` |
| `PYFEM_MODEL_NAME` | 覆盖模型名 | 默认不覆盖 |
| `PYFEM_STEP_NAME` | 显式指定步骤名 | 默认由作业或模型自动解析 |
| `PYFEM_RESULTS_PATH` | 结果 JSON 路径 | JSON 后端会按输入文件名生成默认结果路径 |
| `PYFEM_WRITE_VTK` | 是否导出 VTK | 默认开启 |
| `PYFEM_VTK_PATH` | VTK 导出路径 | 未指定时按输入文件名生成默认 `.vtk` 路径 |
| `PYFEM_REPORT_PATH` | 运行报告 JSON 路径 | 默认不输出报告文件 |

### 6. 使用方法

#### 6.1 源码目录直接运行

当前版本最稳妥的做法是从项目根目录直接运行脚本入口。`run_gui.py` 与 `run_case.py` 都会在启动时把 `src` 目录加入导入路径，因此源码目录直接运行是当前最明确的一条主线。

如果只需要验证 GUI 是否能启动，可先执行：

```bash
python run_gui.py
```

如果只需要验证求解主线是否可运行，可先准备一个 `.inp` 文件，再执行：

```bash
python run_case.py
```

#### 6.2 默认输出路径

脚本和 GUI 都沿用同一套默认命名思路：

- 输入文件 `beam.inp` 的默认结果路径是 `beam.results.json`。
- 如果启用 VTK 导出，默认导出路径是 `beam.vtk`。
- GUI 运行当前模型时，会先写出一个运行快照，再对该快照执行正式求解。

### 7. 输入示例或代码示例

下面是一个仅用环境变量覆盖路径的最小脚本启动示例：

```powershell
$env:PYFEM_INP_PATH = "D:\work\beam.inp"
$env:PYFEM_RESULTS_PATH = "D:\work\beam.results.json"
$env:PYFEM_VTK_PATH = "D:\work\beam.vtk"
$env:PYFEM_WRITE_VTK = "1"
python run_case.py
```

### 8. 输出或结果说明

`run_case.py` 运行成功后，终端至少会输出以下信息中的主要项：

- `model = ...`
- `step = ...`
- `results = ...`
- `frame_count = ...`
- `history_count = ...`
- `vtk = ...`（仅在启用导出时）
- `report = ...`（仅在设置 `PYFEM_REPORT_PATH` 时）

### 9. 限制与注意事项

1. 当前不能把“源码可运行”直接等同于“已经形成正式安装包流程”。
2. 如果直接调用 Python API，而运行环境又不在项目根目录，需要先解决导入路径问题。
3. GUI 3D 预览依赖 `PySide6 + pyvista + pyvistaqt` 的实际可用性；缺失时可能退回 placeholder。
4. 多步骤模型在脚本或 API 中可能需要显式给出 `step_name`。

### 10. 常见错误与排查

| 现象 | 常见原因 | 处理方法 |
| --- | --- | --- |
| `run_case.py` 报输入文件不存在 | `PYFEM_INP_PATH` 未设置或指向错误路径 | 优先检查输入文件绝对路径 |
| `run_case.py` 提示需要显式指定步骤 | 模型或作业中包含多个步骤 | 设置 `PYFEM_STEP_NAME` 或 API 的 `step_name` |
| GUI 启动时报缺少 Qt 相关模块 | GUI 依赖未安装 | 补齐 `PySide6` 等 GUI 依赖 |
| 求解阶段导入失败或线代失败 | `numpy` / `scipy` 未准备好 | 补齐数值依赖后再启动 |

### 11. 相关主题

- 第 1 章 软件概览
- 第 3 章 快速开始
- 后续章节中的“作业、快照与派生算例”
- 后续章节中的“Python API / Facade / Probe”

---

## 第 3 章 快速开始

本章位于“环境已经可启动、需要第一次跑通”的位置。目标不是穷举功能，而是用一个最小算例把“输入文件 -> 求解 -> 结果文件 -> 结果查看”这条链路走通。

### 1. 本教程目标

完成以下最小闭环：

1. 准备一个两节点梁单元静力算例。
2. 通过 `run_case.py` 运行该算例。
3. 得到 `.results.json` 与 `.vtk` 文件。
4. 确认结果文件已生成，并知道下一步可以去哪里查看结果。

### 2. 前置条件

开始前应满足以下条件：

1. 已准备 Python `>=3.11` 环境。
2. 已确认数值依赖可用。
3. 已位于 pyFEM 项目源码根目录，或至少可以正常运行 `run_case.py`。

### 3. 本例模型或任务说明

本例采用当前代码包中的验证性 B21 梁单元静力样例。模型含义如下：

- 一根长度为 `2.0` 的二维梁单元。
- 根部全约束。
- 梁端在 `UY` 方向施加向下集中力。
- 步骤类型为静力线性。

这个样例的意义在于：

- 输入短。
- 支持路径稳定。
- 能同时产出位移、反力、应力相关结果与全局历史量。

### 4. 详细步骤

#### 4.1 准备输入文件

新建 `beam.inp`，写入以下内容：

```inp
*Heading
Task 08 run_case script example
*Node
1, 0.0, 0.0
2, 2.0, 0.0
*Element, type=B21, elset=BEAM_SET
1, 1, 2
*Nset, nset=ROOT
1
*Nset, nset=TIP
2
*Material, name=STEEL
*Elastic
1000000.0, 0.3
*Density
4.0
*Beam Section, elset=BEAM_SET, material=STEEL
0.03, 0.0002
*Step, name=LOAD_STEP
*Static
*Boundary
ROOT, 1, 2, 0.0
ROOT, 6, 6, 0.0
*Cload
TIP, 2, -12.0
*End Step
```

说明：

- 该示例没有显式写 `*Output`。
- 当前 importer 在这种情况下会按步骤类型自动补上默认输出请求。
- 对于静力步骤，默认输出中包含节点位移/反力、单元中心结果、积分点结果、恢复结果、平均结果和时间历史。

#### 4.2 运行脚本

推荐通过环境变量覆盖路径，不直接修改 `run_case.py` 顶部常量。

```powershell
$env:PYFEM_INP_PATH = "D:\work\beam.inp"
$env:PYFEM_RESULTS_PATH = "D:\work\beam.results.json"
$env:PYFEM_VTK_PATH = "D:\work\beam.vtk"
$env:PYFEM_WRITE_VTK = "1"
python run_case.py
```

如果只想使用默认命名，也可以把输入文件命名为 `Job-1.inp` 并放在项目根目录，然后直接执行：

```bash
python run_case.py
```

#### 4.3 检查终端输出

运行成功后，应能在终端看到与下面同类的信息：

```text
model = ...
step = LOAD_STEP
results = ...beam.results.json
frame_count = 1
history_count = ...
vtk = ...beam.vtk
```

#### 4.4 检查结果文件

确认以下文件已经生成：

1. `beam.results.json`
2. `beam.vtk`
3. 可选的运行报告 JSON（仅在设置 `PYFEM_REPORT_PATH` 时生成）

#### 4.5 可选：打开结果

如果 GUI 环境可用，可以启动 GUI 后打开结果文件，继续查看步骤、帧、字段、历史量和 probe。

### 5. 每一步的预期结果

| 步骤 | 预期结果 |
| --- | --- |
| 写入 `beam.inp` | 输入文件可被 `run_case.py` 读取 |
| 执行求解 | 终端输出模型名、步骤名和结果路径 |
| 检查结果文件 | 能看到 `.results.json` 与 `.vtk` |
| 打开结果 | 可以按步骤和字段浏览结果 |

参考说明：

- 该验证样例在代码测试中对应的梁端 `UY` 位移参考值约为 `-0.16`。
- 这条数值可作为当前版本快速自检时的参考，不应替代更完整的工程验证流程。

### 6. 常见出错点

1. 输入文件路径写对了，但运行目录不对，导致脚本实际没有读到目标文件。
2. 模型中如果自行改成多个步骤，却没有显式指定 `step_name`，会触发步骤解析错误。
3. 将 GUI 中可见的预留按钮误当作本教程必需步骤。当前快速开始不依赖这些入口。
4. 误以为没有显式 `*Output` 就不会产生结果。当前 importer 会按步骤类型补默认输出请求。

### 7. 最终结果说明

完成本教程后，你已经走通了当前版本最稳定的一条正式主线：

```text
beam.inp
    -> run_case.py
    -> beam.results.json
    -> beam.vtk
```

如果后续需要继续深入，可以沿两条方向扩展：

1. 保持 INP 主线，继续学习步骤、材料、输出请求和结果字段。
2. 进入 GUI 或 Python API 主线，学习模型浏览、作业提交和结果消费。

### 8. 本例涉及到的核心对象总结

| 对象 | 本例中的角色 |
| --- | --- |
| `B21` | 两节点梁单元 |
| `STEEL` | 线弹性材料定义 |
| `BEAM_SET` | 梁截面所作用的单元集合 |
| `ROOT` | 施加位移边界的节点集 |
| `TIP` | 施加集中载荷的节点集 |
| `LOAD_STEP` | 静力线性分析步骤 |
| 默认输出请求 | 在未显式写 `*Output` 时自动补入的结果请求 |

### 9. 可继续尝试的扩展练习

1. 在保持单步模型的前提下，显式添加 `*Output Request`，观察结果字段是否变化。
2. 将同一条梁样例改为 `modal` 或 `implicit_dynamic`，对比输出变量集合。
3. 用 GUI 打开该模型，再通过 GUI 提交 job，观察快照与结果路径的生成方式。
4. 使用 `PyFEMSession` 运行同一个输入文件，再用 `open_results()` 读取结果。

### 10. 相关主题

- 第 1 章 软件概览
- 第 2 章 安装与启动
- 后续章节中的“输入文件参考”
- 后续章节中的“Procedures 层”
- 后续章节中的“结果层”

## 第 4 章 定义层与编译层

本章只回答两个问题：定义层为什么存在，编译层为什么存在。

它不再继续承担 `kernel`、`procedures`、`solver` 的统一说明职责。换句话说，本章从现在开始只负责“定义对象如何组织”和“定义对象如何被编译为 `CompiledModel`”，至于局部 runtime、过程推进和全局离散求解，将在后续章节分别展开。

### 1. 这一层为什么存在

#### 1.1 定义层为什么存在

定义层解决的是“要算什么”。它保存模型、网格、材料、截面、边界条件、载荷、输出请求、步骤和作业顺序等正式定义对象。

定义层的特点是：

1. 对象可被 INP 导入器、GUI 编辑器和 Python API 直接创建或修改。
2. 对象之间主要通过名称引用关联，而不是通过求解阶段的运行时句柄关联。
3. 定义层对象本身不直接承担求解执行职责。

#### 1.2 编译层为什么存在

编译层解决的是“如何把定义翻译为可执行对象图”。在 pyFEM 当前主线里，这一层至少承担以下职责：

1. 建立 canonical `CompilationScope`。
2. 把部件、本地集合、实例 alias、刚体变换放进统一解析边界。
3. 为节点和其他自由度拥有者建立全局 DOF 编号。
4. 根据定义对象的类型键选择 provider，并生成正式 runtime 或 placeholder runtime。
5. 产出 `CompiledModel`，作为后续 procedures 层与 solver 层消费的编译结果。

这里要特别说明一点：`DofManager` 的源码物理位置位于 `pyfem.kernel.dof_manager`，但在本帮助文档的结构口径中，它承担的是编译桥接职责，因此归入编译层说明。

### 2. 定义对象家族

下表先给出本章涉及的定义对象家族地图。它只回答“有哪些对象家族以及各自承担什么职责”，不展开字段级细节。

| 家族 | 所属层 | 解决的问题 | 核心对象 | 与下游关系 |
| --- | --- | --- | --- | --- |
| 网格记录家族 | 定义层 | 表达节点、单元、集合、表面、方向等部件级记录 | `NodeRecord`、`ElementRecord`、`Surface`、`Orientation` | 被 `Mesh`、`CompilationScope` 和编译器消费 |
| 定义对象家族 | 定义层 | 组织模型、工况、步骤和作业的正式定义 | `ModelDB`、`Part`、`Mesh`、`Assembly`、`PartInstance`、`MaterialDef`、`SectionDef`、`BoundaryDef`、`StepDef`、`JobDef` | 作为 `Compiler.compile()` 的输入 |
| 编译桥接家族 | 编译层 | 解析作用域、自由度和 provider 绑定，并生成编译结果 | `CompilationScope`、`Compiler`、`DofManager`、`CompiledModel`、`RuntimeRegistry`、各类 `*BuildRequest` | 向后续 runtime 主线交付 `CompiledModel` |

#### 2.1 定义对象结构表

下表只回答“这个家族里有哪些核心对象，以及各自承担什么职责”。字段级表格将在第 8 章继续展开。

| 对象 | 对象类型 | 家族内角色 | 主要职责 | 上下游关系 |
| --- | --- | --- | --- | --- |
| `ModelDB` | 容器类 | 定义层顶层数据库 | 统一持有所有正式定义对象并负责校验 | 上游为导入器、GUI、API；下游为 `Compiler` |
| `Part` | 容器类 | 可复用部件容器 | 持有部件级 `Mesh` 与元数据 | 被 `ModelDB.parts` 按名持有 |
| `Mesh` | 容器类 | 部件级网格容器 | 持有节点、单元、集合、表面、方向 | 被 `Part` 直接消费 |
| `Assembly` | 容器类 | 装配组织容器 | 持有 `PartInstance` 与实例级 alias | 为 `CompilationScope` 提供实例边界 |
| `PartInstance` | 记录类 | 装配实例记录 | 通过 `part_name` 指向 `Part`，并提供刚体变换 | 参与形成 canonical scope |
| `MaterialDef` / `SectionDef` | 定义类 | 物理定义对象 | 定义材料类型、截面类型、材料绑定和区域绑定 | 编译为 `MaterialRuntime`、`SectionRuntime` |
| `BoundaryDef` / `NodalLoadDef` / `DistributedLoadDef` / `InteractionDef` | 定义类 | 工况定义对象 | 定义目标、作用域、类型键与参数 | 被步骤和编译器引用 |
| `OutputRequest` | 定义类 | 输出定义对象 | 定义输出变量、目标和频率 | 被 `StepDef` 引用 |
| `StepDef` | 定义类 | 分析步定义对象 | 定义一步分析需要使用的边界、载荷和输出请求 | 被编译为 `ProcedureRuntime` |
| `JobDef` | 定义类 | 作业顺序对象 | 定义步骤执行顺序 | 决定 `CompiledModel.iter_step_names()` 的优先顺序 |

#### 2.2 `ModelDB` 顶层结构

`ModelDB` 是当前定义层的唯一正式入口。源码可直接确认的顶层字段如下。

| 字段 | 数据格式 | 是否可空 | 当前含义 |
| --- | --- | --- | --- |
| `name` | `str` | 否 | 模型名 |
| `metadata` | `Metadata` | 否 | 模型级元数据 |
| `parts` | `dict[str, Part]` | 否 | 部件定义映射 |
| `assembly` | `Assembly | None` | 是 | 装配定义 |
| `materials` | `dict[str, MaterialDef]` | 否 | 材料定义映射 |
| `sections` | `dict[str, SectionDef]` | 否 | 截面定义映射 |
| `boundaries` | `dict[str, BoundaryDef]` | 否 | 边界条件定义映射 |
| `nodal_loads` | `dict[str, NodalLoadDef]` | 否 | 节点载荷定义映射 |
| `distributed_loads` | `dict[str, DistributedLoadDef]` | 否 | 分布载荷定义映射 |
| `interactions` | `dict[str, InteractionDef]` | 否 | 相互作用定义映射 |
| `output_requests` | `dict[str, OutputRequest]` | 否 | 输出请求定义映射 |
| `steps` | `dict[str, StepDef]` | 否 | 分析步定义映射 |
| `raw_keyword_blocks` | `dict[str, RawKeywordBlockDef]` | 否 | 受控 raw keyword / extension block |
| `job` | `JobDef | None` | 是 | 作业定义 |

需要额外记住两点：

1. `ModelDB.constraints` 只是 `boundaries` 的兼容属性，不是独立存储。
2. `ModelDB.procedures` 只是 `steps` 的兼容属性，不是独立存储。

### 3. 作用域与名称引用

#### 3.1 为什么需要 `CompilationScope`

定义层的大多数对象通过名称关联，而名称只有在明确作用域后才有确定含义。`CompilationScope` 的存在，就是为了把“部件本地名”“实例名”“装配 alias”“几何变换后节点记录”组织到统一边界内。

源码可直接确认以下规则：

1. 有装配且存在实例时，一个实例形成一个 canonical scope，`scope_name = instance.name`。
2. 没有装配时，一个部件形成一个 canonical scope，`scope_name = part.name`。
3. 节点几何记录通过 `CompilationScope.get_node_geometry_record()` 在作用域内施加刚体变换。
4. 节点集、单元集和表面会优先命中实例级 alias，未命中时再回退到部件本地对象。

#### 3.2 名称引用表

下表专门说明定义层里最关键的名称引用关系。

| 字段 | 所属对象 | 引用形式 | 解析范围 | 最终命中对象 | 备注 |
| --- | --- | --- | --- | --- | --- |
| `part_name` | `PartInstance` | 字符串名称 | `ModelDB.parts` | `Part` | 装配实例不复制部件 |
| `section_name` | `ElementRecord` | 直接名称 | `ModelDB.sections` | `SectionDef` | 属于单元直接绑定 |
| `material_name` | `SectionDef` | 直接名称 | `ModelDB.materials` | `MaterialDef` | 截面绑定材料 |
| `region_name + scope_name` | `SectionDef` | 区域名 + 作用域名 | canonical scope 对应部件的 `element_sets` | 一组单元名 | 当前正式主线按部件本地 `element_sets` 解析 |
| `target_name + target_type + scope_name` | `BoundaryDef` | 目标名 + 目标类型 + 作用域名 | `CompilationScope.resolve_node_names()` | 节点或节点集 | 当前边界目标类型正式支持 `node`、`node_set` |
| `boundary_names` | `StepDef` | 名称元组 | `ModelDB.boundaries` | `BoundaryDef` 集合 | 步骤定义通过名字消费工况 |
| `nodal_load_names` | `StepDef` | 名称元组 | `ModelDB.nodal_loads` | `NodalLoadDef` 集合 | 同上 |
| `distributed_load_names` | `StepDef` | 名称元组 | `ModelDB.distributed_loads` | `DistributedLoadDef` 集合 | 同上 |
| `output_request_names` | `StepDef` | 名称元组 | `ModelDB.output_requests` | `OutputRequest` 集合 | 同上 |
| `step_names` | `JobDef` | 名称元组 | `ModelDB.steps` | `StepDef` 顺序集合 | 决定作业中的步骤执行顺序 |

#### 3.3 当前需要保守处理的边界

1. `SectionDef.region_name` 当前正式主线按部件本地 `element_sets` 解析，不应扩大写成任意装配级区域名。
2. `ElementRecord.region_name` 字段存在，但主编译路径并不直接依赖它完成正式截面绑定，因此本章不把它写成稳定绑定主线。
3. `BoundaryDef.constraint_type` 是对 `boundary_type` 的兼容属性；文档说明应优先使用定义层口径 `boundary_type`，但在编译层需要知道编译器会通过兼容属性查找 constraint provider。

### 4. 编译桥接对象家族

#### 4.1 这一家族解决什么问题

编译桥接对象家族负责把定义对象翻译成后续层可消费的对象图。它不直接执行分析步骤，也不直接保存结果数据库，但它决定了：

1. 哪些 canonical scope 会被创建。
2. 哪些节点与自由度会获得全局编号。
3. 哪个定义对象命中哪个 provider。
4. 未命中 provider 时是否生成 placeholder runtime。
5. `CompiledModel` 中最终持有哪些 runtime 字典。

#### 4.2 编译桥接对象结构表

| 对象 | 对象类型 | 家族内角色 | 主要职责 | 何时被调用 |
| --- | --- | --- | --- | --- |
| `CompilationScope` | 记录类 / 服务类 | canonical scope 记录 | 提供节点、集合、表面、方向和局部几何解析能力 | 编译元素、边界、载荷时 |
| `Compiler` | 服务类 | 编译入口 | 组织校验、provider 绑定和 runtime 构建 | `compile(model)` 时 |
| `DofManager` | 服务类 | 自由度编号管理器 | 注册、冻结并查询全局 DOF | 编译元素与边界时 |
| `CompiledModel` | 容器类 | 编译产物 | 持有定义层源模型、DOF 管理器、各类 runtime 和步骤状态快照 | 编译完成后被 procedures 层消费 |
| `RuntimeRegistry` | 内部服务类 | provider 查找表 | 按类型键查找 material/section/element/constraint/interaction/procedure provider | 由 `Compiler` 内部使用 |
| `MaterialBuildRequest` 等 `*BuildRequest` | 请求对象 | provider 输入载体 | 将定义对象、作用域、材料、截面、DOF 等编译上下文打包给 provider | 由 `Compiler` 内部构造 |

#### 4.3 编译映射表

下表只说明“对象如何从定义进入编译产物”，不展开 runtime 家族的具体接口。

| 上游对象 | 桥接对象 / 输入 | 下游对象 | 当前正式说明 |
| --- | --- | --- | --- |
| `MaterialDef` | `MaterialBuildRequest` + material provider | `CompiledModel.material_runtimes` | 未命中 provider 时进入 `MaterialRuntimePlaceholder` |
| `SectionDef` | `SectionBuildRequest` + section provider | `CompiledModel.section_runtimes` | 编译时会带入已编好的 `material_runtimes` |
| `ElementRecord` | `CompilationScope` + `ElementBuildRequest` + DOF layout | `CompiledModel.element_runtimes` | 单元坐标、DOF 索引和材料/截面引用在此阶段确定 |
| `BoundaryDef` | `ConstraintBuildRequest` + `DofManager` + target scope 解析 | `CompiledModel.constraint_runtimes` | 先解析受约束 DOF，再构造 constraint runtime |
| `InteractionDef` | `InteractionBuildRequest` | `CompiledModel.interaction_runtimes` | 未命中 provider 时进入 placeholder |
| `StepDef` | `ProcedureBuildRequest` | `CompiledModel.step_runtimes` | `procedure_type` 决定步骤运行时类型 |
| `JobDef` | `CompiledModel.iter_step_names()` | 步骤执行顺序 | 若存在 `job.step_names`，优先以其顺序为准 |

### 5. 核心类说明

#### 5.1 `ModelDB`

- 所属层：定义层
- 家族：定义对象家族
- 职责：作为问题定义唯一来源，统一持有部件、工况、步骤与作业定义，并在编译前执行一致性校验。
- 核心字段：`parts`、`assembly`、`materials`、`sections`、`boundaries`、`nodal_loads`、`distributed_loads`、`interactions`、`output_requests`、`steps`、`job`
- 上下游关系：上游为 INP 导入器、GUI 与 API；下游为 `Compiler.compile()`。
- 典型误解：它不是“已经可运行的模型”，必须先经过编译。

下表只列当前建议作为正式公开边界理解的方法。

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 |
| --- | --- | --- | --- | --- | --- |
| `add_part()` | `Part` | `None` | 注册部件定义 | 组装模型时 | 导入器、GUI、API |
| `get_part()` | `part_name` | `Part` | 按名取部件 | 编译或编辑时 | `Compiler`、上层工具 |
| `set_assembly()` | `Assembly` | `None` | 设置装配定义 | 构造装配模型时 | 导入器、GUI、API |
| `add_material()` / `add_section()` / `add_boundary()` / `add_step()` | 各类定义对象 | `None` | 注册正式定义对象 | 建模时 | 导入器、GUI、API |
| `iter_compilation_scopes()` | 无 | `tuple[CompilationScope, ...]` | 生成 canonical scope 集合 | 编译时 | `Compiler` |
| `iter_target_scopes()` | `scope_name | None` | `tuple[CompilationScope, ...]` | 解析目标作用域集合 | 解析边界或载荷目标时 | `Compiler` |
| `resolve_compilation_scope()` | `scope_name` | `CompilationScope | None` | 按名解析单个作用域 | 高级编译桥接时 | 编译服务 |
| `validate()` | 无 | `None` | 校验定义关系完整性 | 编译前 | `Compiler` |

#### 5.2 `CompilationScope`

- 所属层：编译层
- 家族：编译桥接对象家族
- 职责：描述一个 canonical scope，并在该作用域下解析节点、单元、集合、表面、方向和几何变换结果。
- 核心字段：`scope_name`、`part_name`、`part`、`transform`、`instance_name`、`node_sets`、`element_sets`、`surfaces`
- 主要边界：它是桥接对象，不是结果对象，也不是 kernel runtime。
- 典型误解：它不等于 `PartInstance`，而是由部件和实例信息推导出的编译期作用域对象。

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 |
| --- | --- | --- | --- | --- | --- |
| `qualify_node_name()` | `node_name` | `str` | 生成作用域内 canonical 节点名 | 需要规范名时 | 编译服务 |
| `qualify_element_name()` | `element_name` | `str` | 生成作用域内 canonical 单元名 | 需要规范名时 | 编译服务 |
| `get_node_geometry_record()` | `node_name` | `NodeRecord` | 返回施加实例变换后的节点记录 | 编译单元时 | `Compiler` |
| `iter_node_geometry_records()` | 无 | `tuple[NodeRecord, ...]` | 遍历当前作用域全部几何节点 | 几何消费时 | 编译服务 |
| `resolve_node_names()` | `target_type`、`target_name` | `tuple[str, ...]` | 在作用域内解析节点目标 | 解析边界目标时 | `Compiler` |
| `resolve_element_names()` | `target_type`、`target_name` | `tuple[str, ...]` | 在作用域内解析单元目标 | 解析区域目标时 | 编译服务 |
| `get_surface()` | `surface_name` | `Surface | None` | 取实例级或部件级表面 | 解析分布载荷目标时 | 编译服务 |
| `get_orientation()` | `orientation_name` | `Orientation | None` | 返回经过实例旋转映射后的方向定义 | 编译单元时 | `Compiler` |
| `has_part_element_region()` | `region_name` | `bool` | 判断部件本地是否存在单元区域 | 区域截面匹配时 | `Compiler` |
| `resolve_part_element_region()` | `region_name` | `tuple[str, ...]` | 仅按部件本地解析单元区域 | 区域截面匹配时 | `Compiler` |

#### 5.3 `Compiler`

- 所属层：编译层
- 家族：编译桥接对象家族
- 职责：总控编译流程，依次构建材料、截面、单元、约束、相互作用和步骤运行时，并在完成后冻结 `DofManager`。
- 核心字段：`_registry`
- 主要方法：当前正式公开方法很少，主入口是 `compile()`；其余 `_build_*`、`_resolve_*` 为内部编译流程。
- 典型误解：它不执行分析步骤，只负责生成 `CompiledModel`。

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 |
| --- | --- | --- | --- | --- | --- |
| `compile()` | `ModelDB` | `CompiledModel` | 校验模型、构建所有 runtime 字典并冻结自由度布局 | 正式编译入口 | API、作业入口、测试 |

#### 5.4 `DofManager`

- 所属层：文档口径归入编译层
- 家族：编译桥接对象家族
- 职责：为节点和其他自由度拥有者分配全局编号，并提供后续查询接口。
- 核心状态：`_index_by_location`、`_locations_by_index`、`_dof_names_by_owner`、`_is_finalized`
- 主要方法：当前以注册、查询、冻结为主。
- 典型误解：它不是 solver 的求解器对象，也不是结果对象。

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 |
| --- | --- | --- | --- | --- | --- |
| `register_dof()` | `DofLocation` | `int` | 注册单个 DOF | 编译阶段 | `Compiler` |
| `register_node_dofs()` | `NodeLocation`、`dof_names` | `tuple[int, ...]` | 为节点批量注册自由度 | 编译单元时 | `Compiler` |
| `register_owned_dofs()` | `scope_name`、`owner_kind`、`owner_name`、`dof_names` | `tuple[int, ...]` | 为任意拥有者注册 DOF | 编译扩展对象时 | 编译服务 |
| `register_extra_dofs()` | `scope_name`、`owner_name`、`dof_names` | `tuple[int, ...]` | 兼容旧接口的附加 DOF 注册 | 兼容场景 | 编译服务 |
| `get_global_id()` | `DofLocation` | `int` | 查询单个 DOF 全局编号 | 编译边界或载荷时 | `Compiler` |
| `get_node_dof_ids()` | `NodeLocation`、`dof_names | None` | `tuple[int, ...]` | 查询节点 DOF 编号 | 约束或载荷解析时 | 编译服务 |
| `get_owner_dof_ids()` | `scope_name`、`owner_kind`、`owner_name`、`dof_names | None` | `tuple[int, ...]` | 查询任意拥有者 DOF 编号 | 同上 | 编译服务 |
| `iter_descriptors()` | 无 | `tuple[DofDescriptor, ...]` | 按全局顺序遍历已注册 DOF | 调试或检查时 | 上层工具 |
| `num_dofs()` / `count()` | 无 | `int` | 返回当前 DOF 总数 | 编译完成后 | 上层工具 |
| `finalize()` | 无 | `None` | 冻结布局，禁止继续注册 | 编译末尾 | `Compiler` |

#### 5.5 `CompiledModel`

- 所属层：编译层
- 家族：编译桥接对象家族
- 职责：持有 `ModelDB`、`DofManager`、各类 runtime 字典，以及跨步骤继承所需的 `step_state_snapshots`。
- 核心字段：`model`、`dof_manager`、`element_runtimes`、`material_runtimes`、`section_runtimes`、`constraint_runtimes`、`interaction_runtimes`、`step_runtimes`、`step_state_snapshots`
- 上下游关系：上游为 `Compiler`；下游主要由 procedures 层消费。
- 典型误解：它不是 solver 的 `DiscreteProblem`，也不是结果数据库。

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 |
| --- | --- | --- | --- | --- | --- |
| `model_name` | 无 | `str` | 返回编译模型名 | 查询编译结果时 | 上层工具 |
| `source_model` | 无 | `ModelDB` | 返回源定义模型 | 兼容旧接口或回查定义时 | 上层工具 |
| `procedure_runtimes` | 无 | `dict[str, ProcedureRuntime]` | 返回步骤运行时字典的兼容视图 | 兼容旧接口时 | 上层工具 |
| `get_element_runtime()` | `qualified_name` | `ElementRuntime` | 按 canonical 名称取单元 runtime | 调试或高级扩展时 | procedures / 工具层 |
| `get_step_runtime()` | `step_name` | `ProcedureRuntime` | 按步骤名取过程 runtime | 执行某一步前 | 作业入口、procedures 层 |
| `get_procedure()` | `step_name` | `ProcedureRuntime` | `get_step_runtime()` 的兼容入口 | 兼容旧接口时 | 上层工具 |
| `publish_step_state()` | `step_name`、`channel`、`state` | `None` | 发布可被后续步骤继承的 committed 状态 | 步骤完成后 | procedures 层 |
| `resolve_inherited_step_state()` | `step_name`、`channel` | `RuntimeState | None` | 解析当前步骤可继承的最近一步状态 | 多步执行前 | procedures 层 |
| `iter_step_names()` | 无 | `Iterable[str]` | 返回正式步骤顺序 | 多步执行或作业调度时 | 作业入口、procedures 层 |

### 6. provider 与 placeholder runtime

这部分只解释编译层如何处理 provider 绑定，不代替后续的 runtime 家族章节。

#### 6.1 provider 在编译层中的含义

在当前源码主线中，`Compiler` 会通过 `RuntimeRegistry` 查找以下 provider：

- material provider：按 `MaterialDef.material_type`
- section provider：按 `SectionDef.section_type`
- element provider：按 `ElementRecord.type_key`
- constraint provider：按 `BoundaryDef.constraint_type`，其值来自 `boundary_type` 的兼容属性
- interaction provider：按 `InteractionDef.interaction_type`
- procedure provider：按 `StepDef.procedure_type`

这里的 provider 绑定，回答的是“当前定义对象应该被编译成哪一种正式 runtime 实现”。

#### 6.2 placeholder runtime 在编译层中的含义

如果某个类型键没有命中 provider，当前编译主线不会凭空虚构正式实现，而是生成对应的 placeholder runtime，例如：

- `MaterialRuntimePlaceholder`
- `SectionRuntimePlaceholder`
- `ElementRuntimePlaceholder`
- `ConstraintRuntimePlaceholder`
- `InteractionRuntimePlaceholder`
- `StepRuntimePlaceholder`

本帮助文档对 placeholder runtime 的口径保持保守：

1. 它们表示“编译结构上保留了该对象的位置”。
2. 它们不应被写成稳定、可推荐依赖的正式用户接口。
3. 只要源码和主线文档没有明确说明某类 placeholder 已形成完整能力，就必须把相关能力视为待确认或未纳入正式支持边界。

### 7. 对象创建、引用与消费示例

#### 7.1 构造示例

下面的例子只演示定义对象如何进入 `ModelDB`，不展开求解。

```python
mesh = Mesh()
mesh.add_node(NodeRecord(name="N1", coordinates=(0.0, 0.0)))
mesh.add_node(NodeRecord(name="N2", coordinates=(1.0, 0.0)))
mesh.add_element(
    ElementRecord(
        name="E1",
        type_key="B21",
        node_names=("N1", "N2"),
        section_name="BEAM_SEC",
    )
)
mesh.add_element_set("ALL", ("E1",))
mesh.add_node_set("FIXED", ("N1",))

part = Part(name="BeamPart", mesh=mesh)

model = ModelDB(name="beam-demo")
model.add_part(part)
model.add_material(
    MaterialDef(
        name="Steel",
        material_type="linear_elastic",
        parameters={"E": 210000.0, "nu": 0.3},
    )
)
model.add_section(
    SectionDef(
        name="BEAM_SEC",
        section_type="beam",
        material_name="Steel",
        region_name="ALL",
        scope_name="BeamPart",
    )
)
```

这个例子里，`NodeRecord` 先进入 `Mesh`，`Mesh` 再进入 `Part`，`Part` 再进入 `ModelDB.parts`；`SectionDef` 则通过 `ModelDB.add_section()` 进入定义层顶层字典。

#### 7.2 引用示例

下面的例子只演示“名字如何命中对象”。

```python
boundary = BoundaryDef(
    name="BC-FIX",
    target_name="FIXED",
    target_type="node_set",
    scope_name="BeamPart",
    dof_values={"UX": 0.0, "UY": 0.0, "RZ": 0.0},
)

step = StepDef(
    name="Step-1",
    procedure_type="static",
    boundary_names=("BC-FIX",),
)
```

这里有两条连续引用链：

1. `ElementRecord.section_name = "BEAM_SEC"` 会命中 `ModelDB.sections["BEAM_SEC"]`。
2. `BoundaryDef.target_name = "FIXED"` 需要与 `target_type = "node_set"` 和 `scope_name = "BeamPart"` 一起解释，最终在 `CompilationScope.resolve_node_names()` 中命中当前作用域下的节点集。

#### 7.3 消费示例

下面的例子演示编译之后对象如何进入 `CompiledModel`。

```python
compiled = Compiler().compile(model)

beam_section = compiled.section_runtimes["BEAM_SEC"]
step_runtime = compiled.get_step_runtime("Step-1")
dof_total = compiled.dof_manager.num_dofs()
```

这说明：

1. `Compiler.compile()` 的直接产物是 `CompiledModel`。
2. 定义层对象不会原样“执行”，而是先被编译为 runtime 字典。
3. procedures 层后续消费的入口不是 `StepDef`，而是 `CompiledModel.step_runtimes` 中的 `ProcedureRuntime`。

### 8. 本章边界对照

下表用于澄清本章最容易被混写的相邻对象。

| 对象或边界对 | 所属层 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- | --- |
| `PartInstance` | 定义层 | 装配中的实例记录 | canonical scope | `CompilationScope` 是编译期派生对象 |
| `CompilationScope` | 编译层 | 解析作用域与几何变换的桥接对象 | 部件定义对象 | 它不持有顶层模型数据库 |
| `Compiler` | 编译层 | 编译入口服务 | 分析步骤执行器 | 它构造 runtime，但不运行步骤 |
| `DofManager` | 编译层口径 | DOF 编号服务 | 线性求解器 | 只负责编号与查询 |
| `CompiledModel` | 编译层 | 编译结果容器 | solver / 结果数据库 | 它向下游交付对象图 |
| placeholder runtime | 编译层边界对象 | provider 缺席时的占位结构 | 稳定正式能力 | 不应被直接写成“已支持” |

### 9. 本章主线图

```text
定义对象家族
    -> 名称引用与 canonical scope
    -> Compiler / DofManager
    -> CompiledModel
    -> kernel / procedures / solver 后续章节
```

## 第 5 章 输入文件参考

本章位于“已经理解对象关系、开始正式写输入文件”的位置，用于说明当前 importer/exporter 真正支持的 INP 子集。

### 1. 功能

当前 INP 主线负责两件事：

1. 把 INP 文本翻译为 `ModelDB`
2. 把正式支持子集的 `ModelDB` 导出为 INP

本章只描述当前 importer/exporter 能稳定识别和回写的关键字子集。

### 2. 所属类别

本章属于“输入格式参考页”，关注的是关键字、选项、数据行和默认行为，而不是数值理论本身。

### 3. 基本解析规则

当前解析器使用以下基本规则：

1. 以 `*` 开头的行为关键字行。
2. 空行与 `**` 注释行会被忽略。
3. 关键字名称会在解析时转为大写。
4. 关键字选项按逗号分隔；带 `=` 的项会被解析为选项字典。

### 4. 当前支持的顶层关键字

| 顶层关键字 | 当前作用 | 说明 |
| --- | --- | --- |
| `*Heading` | 设置模型描述 | 可用于保存说明文本 |
| `*Preprint` | 允许出现 | 当前作为控制关键字忽略 |
| `*System` | 仅接受空控制块 | 带坐标变换数据行的 `SYSTEM` 当前不在正式 importer 主线 |
| `*Part` / `*End Part` | 定义部件块 | 部件块内有独立支持子集 |
| `*Assembly` / `*End Assembly` | 定义装配块 | 当前只支持 `INSTANCE/NSET/ELSET/SURFACE` 子集 |
| `*Node` | 定义节点 | 可用于 legacy 单部件主线或 `PART` 块内 |
| `*Element` | 定义单元 | 当前依赖正式单元类型子集 |
| `*Nset` | 定义节点集 | 支持普通成员行和 `generate` |
| `*Elset` | 定义单元集 | 支持普通成员行和 `generate` |
| `*Material` | 定义材料入口 | 后续由 `*Elastic`、`*Plastic`、`*Density` 补充参数 |
| `*Elastic` | 定义线弹性参数 | 需要位于 `*Material` 后 |
| `*Plastic` | 定义 J2 塑性参数 | 当前归一化到 `j2_plasticity` |
| `*Density` | 定义密度 | 当前作为材料参数写入 |
| `*Solid Section` | 定义实体或平面截面 | 可显式给 formulation，也可按单元类型推断 |
| `*Beam Section` | 定义梁截面 | 当前读取面积、惯性矩与可选 shear factor |
| `*Surface` | 定义表面 | 当前仅支持 `type=ELEMENT` |
| `*Orientation` | 定义方向 | 当前仅支持 rectangular 一阶轴向定义 |
| `*Boundary` | 定义边界或边界块 | 支持结构化写法和传统数据行写法 |
| `*Cload` | 定义节点载荷或载荷块 | 支持结构化写法和传统数据行写法 |
| `*Dsload` | 定义分布载荷 | 顶层仅支持结构化定义；STEP 内另有传统短写格式 |
| `*Output Request` | 定义正式输出请求对象 | 可在顶层定义为复用对象 |
| `*Raw Keyword` | 定义受控原始关键字块 | 进入 `RawKeywordBlockDef` |
| `*Job` | 定义作业与步骤顺序 | 数据行列出步骤名 |
| `*Step` / `*End Step` | 定义分析步骤 | STEP 内有独立支持子集 |

除上述关键字外，当前顶层出现其他关键字时，导入器会按不支持处理。

### 5. `PART` 与 `ASSEMBLY` 子集

#### 5.1 `PART` 块当前支持的内容

当前 `PART` 块内支持：

- `*Node`
- `*System`（仅空控制块）
- `*Element`
- `*Nset`
- `*Elset`
- `*Surface`
- `*Orientation`
- `*Solid Section`
- `*Beam Section`

当前 `PART` 块内不应写成正式支持的内容包括未在导入器中显式处理的其他 Abaqus 关键字。

#### 5.2 `ASSEMBLY` 块当前支持的内容

当前 `ASSEMBLY` 块只支持：

- `*Instance`
- `*Nset`
- `*Elset`
- `*Surface`

其中：

- `*Instance` 当前表示部件实例与实例刚体变换。
- 装配级 `NSET/ELSET/SURFACE` 当前必须显式给出 `instance=`，用于建立实例级别名。

#### 5.3 实例作用域规则

在有装配实例的模型里，目标对象不能再模糊地依赖部件名。

当前推荐写法是：

```inp
*Boundary
left.ROOT, 1, 2, 0.0
```

如果使用装配别名，也必须保证该别名在当前装配中只唯一对应一个实例；否则导入器无法安全推断作用域。

### 6. 截面、表面与方向关键字

#### 6.1 `*Solid Section`

当前 `*Solid Section` 支持两种模式：

1. 显式通过 `formulation` 或 `section_type` 指定 `solid`、`plane_stress` 或 `plane_strain`
2. 在未显式指定时按区域内首个单元类型推断

当前默认推断规则为：

- `C3D8` -> `solid`
- `CPS4` -> `plane_stress`

如果是平面截面，当前会记录 `thickness`；未提供时默认按 `1.0` 处理。

#### 6.2 `*Beam Section`

当前 `*Beam Section` 会读取：

- `area`
- `moment_inertia_z`
- 可选 `shear_factor`

#### 6.3 `*Surface`

当前 `*Surface` 只支持 `type=ELEMENT`。表面数据行需要至少给出：

1. 区域名或单元名
2. 局部面标签

#### 6.4 `*Orientation`

当前 `*Orientation` 只支持 rectangular 方向定义，且必须给出 `axis_1` 与 `axis_2`。

### 7. `STEP` 块当前支持的关键字

#### 7.1 过程关键字

当前 `STEP` 块内正式支持以下过程关键字：

| 关键字 | 当前归一化后的过程类型 | 说明 |
| --- | --- | --- |
| `*Static` | `static_linear` | 可携带普通步骤参数选项 |
| `*Static Nonlinear` | `static_nonlinear` | 可携带非线性步骤参数选项 |
| `*Frequency` | `modal` | 可读取 `num_modes` |
| `*Dynamic` | `implicit_dynamic` | 可读取 `time_step`、`total_time` 等参数 |

#### 7.2 STEP 内对象关键字

当前 `STEP` 块内支持：

- `*Boundary`
- `*Cload`
- `*Dsload`
- `*Boundary Ref`
- `*Cload Ref`
- `*Dsload Ref`
- `*Output Request`
- `*Output Request Ref`
- `*Restart`
- `*Output`
- `*Step Parameter`
- `*Raw Keyword`

其中有三个特别重要的边界：

1. `*Restart` 当前只接受不带数据行的控制形式。
2. `*Output` 当前只接受 `variable=PRESELECT` 且不带数据行的控制形式。
3. 任何不在上表中的 STEP 子关键字，当前都不应写成正式支持。

#### 7.3 STEP 内边界与载荷的两种写法

`*Boundary`、`*Cload`、`*Dsload` 当前都支持“定义对象”和“直接数据行”两种用法，但边界不同：

| 关键字 | 结构化定义 | 传统数据行写法 | 当前边界 |
| --- | --- | --- | --- |
| `*Boundary` | 支持 | 支持 | 结构化时写成命名对象；传统写法会在导入时生成匿名编号对象 |
| `*Cload` | 支持 | 支持 | 与 `Boundary` 类似 |
| `*Dsload` | 支持 | STEP 内支持；顶层不支持 | 传统短写当前只支持压力代码 `P` |

#### 7.4 顶层定义与步骤引用的关系

当前导入规则里需要区分“定义对象”和“让步骤使用对象”：

- 顶层结构化 `*Boundary` / `*Cload` / `*Dsload` 主要用于定义可复用对象。
- `STEP` 中通过 `*Boundary Ref` / `*Cload Ref` / `*Dsload Ref` 才表示当前步骤要使用这些命名对象。
- 顶层传统写法生成的全局边界和全局节点载荷，会被作为继承对象带入后续步骤。

因此，不能把“对象已经在顶层定义”直接理解为“当前步骤一定会执行它”。

### 8. 输出请求与默认行为

#### 8.1 `*Output Request`

当前正式输出请求关键字为：

```inp
*Output Request, name=..., request_mode=..., target_type=..., position=..., frequency=...
```

数据行写变量名列表，例如：

```inp
U, RF, U_MAG
```

可选项当前包括：

- `name`
- `request_mode`
- `target_type`
- `target`
- `scope`
- `position`
- `frequency`

#### 8.2 `*Output Request Ref`

如果输出请求已经在顶层定义，当前可以在步骤中使用 `*Output Request Ref` 引用名称。

#### 8.3 默认输出请求

如果某个步骤没有显式输出请求，当前 importer 会按步骤类型自动补默认输出请求。

默认规则如下：

- `modal`：默认补 `MODE_SHAPE` 与 `FREQUENCY`
- `implicit_dynamic`：默认补节点位移 `U` 与时间历史 `TIME`
- 其他当前正式过程：默认补节点位移/反力/位移模、单元中心、积分点、恢复、平均和时间历史

这也是为什么很多最小 INP 示例即使没有显式写 `*Output` 或 `*Output Request`，仍然会得到结果字段。

### 9. `RAW KEYWORD` 与 `STEP PARAMETER`

#### 9.1 `*Raw Keyword`

当前 `*Raw Keyword` 会被翻译为受控的 `RawKeywordBlockDef`，而不是直接无约束地拼接任意原始文本。

其当前关键信息包括：

- `name`
- `keyword`
- `placement`
- `step`
- `order`
- `description`
- 以 `raw_` 前缀给出的原始选项

#### 9.2 `*Step Parameter`

`*Step Parameter` 用于保存未被过程关键字专门消费的步骤参数。当前支持：

- 文本
- JSON
- 数值
- 整数
- 布尔值

这类参数会进入 `StepDef.parameters`。

### 10. 正确示例

#### 10.1 最小 legacy 示例

```inp
*Heading
beam example
*Node
1, 0.0, 0.0
2, 2.0, 0.0
*Element, type=B21, elset=BEAM_SET
1, 1, 2
*Nset, nset=ROOT
1
*Nset, nset=TIP
2
*Material, name=STEEL
*Elastic
1000000.0, 0.3
*Beam Section, elset=BEAM_SET, material=STEEL
0.03, 0.0002
*Step, name=LOAD_STEP
*Static
*Boundary
ROOT, 1, 2, 0.0
ROOT, 6, 6, 0.0
*Cload
TIP, 2, -12.0
*End Step
```

#### 10.2 装配作用域示例

```inp
*Assembly, name=ASM
*Instance, name=left, part=BEAM
*End Instance
*Instance, name=right, part=BEAM
5.0, 0.0
*End Instance
*End Assembly
*Step, name=LOAD_STEP
*Static
*Boundary
left.ROOT, 1, 2, 0.0
*Cload
right.TIP, 2, -12.0
*End Step
```

### 11. 错误示例

#### 11.1 在装配模型里把部件名当成实例名

```inp
*Boundary
BEAM.ROOT, 1, 2, 0.0
```

在有装配实例的模型中，这种写法当前不应视为正式支持。目标应优先写成实例作用域。

#### 11.2 使用当前未纳入正式主线的 `SYSTEM` 坐标定义

```inp
*System
0.0, 0.0, 0.0, 1.0, 0.0, 0.0
```

当前 `SYSTEM` 仅允许空控制块。

#### 11.3 在传统 `*Dsload` 中使用非压力代码

```inp
*Dsload
PRESSURE_FACE, TRVEC, -2.0
```

当前传统 `*Dsload` 短写只支持 `P` 压力代码。

### 12. 当前限制

1. 顶层未列入翻译器分支的关键字，当前都不应写成正式支持。
2. `ASSEMBLY` 块当前只支持 `INSTANCE/NSET/ELSET/SURFACE`。
3. `SYSTEM` 仅支持空控制关键字。
4. `OUTPUT` 当前仅支持 `variable=PRESELECT` 的控制形式。
5. `DSLOAD` 的传统短写格式只在 `STEP` 内支持，且仅支持压力代码 `P`。
6. `SURFACE` 当前仅支持 `ELEMENT` 类型。
7. `ORIENTATION` 当前仅支持一阶 rectangular 方向定义。

### 13. 相关主题

- 第 3 章 快速开始
- 第 4 章 定义层与编译层
- 第 6 章 GUI 工作台
- 后续章节中的“建模对象说明”
- 后续章节中的“Procedures 层”

---

## 第 6 章 GUI 工作台

本章位于“已经可以跑通输入文件，希望通过工作台组织建模与结果浏览”的位置，用于说明当前 GUI 骨架、正式主线和预留入口之间的边界。

### 1. 功能概述

当前 GUI 工作台负责把以下动作组织到统一窗口中：

1. 打开模型与查看模型摘要
2. 浏览材料、截面、载荷、边界、步骤和输出请求
3. 通过正式对话框编辑已纳入主线的对象
4. 通过快照主线提交 job
5. 打开结果、浏览步骤/帧/字段/历史/摘要
6. 进行 probe、设置图例、导出 VTK

### 2. 适用范围

本章只描述当前已经形成正式主线的 GUI 能力。

本章不把以下内容写成已完成工作流：

- 接触、约束、耦合、tie 创建
- 幅值管理
- 网格划分与生成
- 完整优化工作流
- 丰富结果字段选择器
- 截图或图形导出工作流
- 正式 Python 控制台集成

### 3. 工作台总体结构

当前主窗口由以下几块组成：

| 区域 | 当前作用 |
| --- | --- |
| 左侧 Navigation | 浏览模型树和结果树 |
| 中左 Module Toolbox | 按模块组织动作按钮 |
| 中央 Viewport | 显示网格、位移形状、等值图或 placeholder |
| 底部 Message Console | 显示消息日志与占位 Python Console |
| 右侧可选 Postprocessing Dock | 显示 Results 与 Inspector |
| 顶部 IO Toolbar | 输入、结果、VTK 和运行入口 |
| 顶部 Context Toolbar | 模块、步骤、帧、字段、显示模式和 probe 关联控件 |
| 底部 Status Bar | 当前任务、模型与结果状态 |

### 4. 模块分区

当前模块顺序固定为：

1. `Part`
2. `Property`
3. `Assembly`
4. `Step`
5. `Interaction`
6. `Load`
7. `Mesh`
8. `Optimization`
9. `Job`
10. `Visualization`

这十个模块并不表示每个模块都已形成同样完整的工作流。

当前应理解为：

- `Property`、`Step`、`Load`、`Job`、`Visualization` 是正式主线更清晰的区域。
- `Interaction`、`Mesh`、`Optimization` 中仍有较多预留入口。

### 5. 模型浏览主线

#### 5.1 导航树

模型加载后，左侧导航树当前会按以下分类显示对象：

- Models
- Parts
- Materials
- Sections
- Assembly
- Steps
- Loads
- BCs
- Output Requests

这使得当前 GUI 的首要角色更接近“模型工作台 + 结果浏览器”，而不是完整的几何建模系统。

#### 5.2 当前已纳入正式编辑主线的对象

GUI 当前明确纳入正式参数编辑主线的对象种类包括：

- `material`
- `section`
- `boundary`
- `nodal_load`
- `distributed_load`
- `output_request`
- `step`
- `instance`

对应的正式编辑对话框或管理器包括：

- `MaterialEditDialog`
- `SectionEditDialog`
- `BoundaryEditDialog`
- `LoadEditDialog`
- `OutputRequestEditDialog`
- `StepEditDialog`
- `InstanceTransformDialog`
- `MaterialManagerDialog`
- `SectionManagerDialog`
- `LoadManagerDialog`
- `BoundaryManagerDialog`
- `StepManagerDialog`

#### 5.3 当前正式模型编辑主线

当前比较稳定的 GUI 建模路径是：

```text
Open Model
    -> 检查导航树与模型摘要
    -> 编辑材料 / 截面 / 边界 / 载荷 / 步骤 / 输出请求
    -> 可选做实例变换与截面分配
    -> Write INP 或 Run Current Model
```

### 6. 作业与快照主线

#### 6.1 当前运行方式

GUI 当前提交作业时，不是额外维护一条与脚本完全无关的私有求解通道，而是先写出正式 snapshot，再通过 snapshot 主线执行。

这条链路可以概括为：

```text
当前活模型
    -> write snapshot
    -> run snapshot
    -> results.json / optional vtk
    -> ResultsFacade
```

#### 6.2 Job 区当前主命令

当前 `Job` 模块中已形成主线的命令包括：

- `Run Current Model`
- `Run Last Snapshot`
- `Open Snapshot Manifest`
- `Open Job Center`
- `Open Current Job Monitor`
- `Open Job Diagnostics`
- `Results / Output...`

#### 6.3 派生算例

`Assembly` / `Optimization` 区域当前还共享一条正式入口：

- `Save As Derived Case`

这条路径会把当前模型另存为新的来源文件，并切换 GUI 当前来源路径。

### 7. 结果主线

#### 7.1 结果打开与结果树

当前 GUI 打开结果后，会基于 `ResultsFacade` 构建结果浏览上下文，并在导航树中组织：

- 步骤
- 帧
- 字段
- 历史
- 摘要

右侧 `ResultsBrowser` 会同步显示步骤、帧、字段、历史量和摘要，并提供详情区。

#### 7.2 当前显示模式

当前工作台支持以下显示模式：

- `Undeformed`
- `Deformed`
- `Contour on Deformed`

如果当前已经选中结果字段，系统会优先将其作为等值图字段处理；否则视图区会保留网格或形状显示。

#### 7.3 当前 probe 种类

当前 probe 对话框支持以下 probe 类型：

- Node
- Element
- Integration Point
- Averaged

probe 的可用字段不是任意组合，而是按当前字段族与结果位置做兼容性匹配。例如：

- 位移族主要对应节点 probe
- 应力族可对应单元、积分点或平均节点 probe
- 某些字段族在当前 probe 类型下不可用时，GUI 会按不兼容处理

#### 7.4 当前结果导出

当前结果侧已形成正式主线的导出能力包括：

- `Open Results`
- `Export VTK`
- `Probe`
- `Legend Settings`
- 结果输出入口中的 report / manifest / results 路径浏览

### 8. 视图区与预览退化路径

#### 8.1 正常情况

如果 `PySide6`、`pyvista` 与 `pyvistaqt` 都可用，且当前会话满足 3D 预览条件，视图区会创建 PyVista 预览控件，显示：

- 网格
- 变形形状
- 等值图
- 图例和颜色映射

#### 8.2 退化情况

如果当前会话不满足预览条件，GUI 不会把这件事伪装成求解失败，而是保留 placeholder 模式。

当前可能进入 placeholder 的典型情况包括：

- `PyVista` / `pyvistaqt` 不可用
- Qt 绑定与 `PySide6` 不一致
- 当前运行环境属于 offscreen / minimal 等安全模式
- 本会话显式禁用 3D preview

因此，当前版本里“没有 3D 预览”并不等于“结果不可用”。结果树、结果浏览器、probe 和 VTK 导出仍然可能正常工作。

### 9. 当前限制与预留入口

#### 9.1 明确仍为 placeholder 的入口

当前界面中可见但仍属于预留入口的典型功能包括：

- `Create Contact`
- `Create Constraint`
- `Create Coupling`
- `Create Tie`
- `Amplitudes...`
- `Generate Mesh`
- `Parameter Variant`
- `Result Field Selector`
- `Screenshot/Export Figure`

此外，`Mesh`、`Optimization` 等模块中的若干按钮当前也仍以 reserved entry 的形式存在。

#### 9.2 GUI 显示层与正式结果字段的区别

GUI 结果显示层会把正式字段组织成“位移、应力、应变、等效应力、主应力”等族，并可能显示“常用”或变体别名。

但这不等于这些显示名都是正式结果字段名。

一个典型例子是：

- GUI 显示层包含 `E_AVG` 族别名
- 当前正式输出与后处理主线并未生成 `E_AVG` 作为正式字段

因此，编写用户帮助时必须区分：

1. GUI 展示层的显示标签
2. 结果系统真正存在的正式字段

### 10. 使用方法

下面给出当前 GUI 最小正式工作流：

1. 启动 `run_gui.py`
2. 打开模型文件
3. 在导航树确认材料、截面、边界、载荷、步骤和输出请求
4. 在正式编辑对话框中修改需要的对象
5. 通过 `Run` 或 `Run Current Model` 提交作业
6. 打开结果
7. 在 `Visualization` 中切换字段、显示模式和 probe
8. 如有需要，导出 VTK

### 11. 输入示例或操作示例

一个最小 GUI 操作顺序可以写成：

```text
Open Model
    -> Property: Materials / Sections
    -> Load: Loads / Boundaries
    -> Step: Steps / Output Controls
    -> Job: Run
    -> Visualization: Open Results / Probe / Export VTK
```

### 12. 输出或结果说明

当前 GUI 成功运行后，常见可见结果包括：

- 模型摘要更新
- 最近一次 job 记录
- 快照 manifest
- `.results.json` 路径
- 可选 `.vtk` 路径
- 结果树中的步骤、帧、字段、历史和摘要

### 13. 限制与注意事项

1. 不能因为按钮可见，就默认该功能已经形成正式用户工作流。
2. `Visualization` 中显示的字段族标签，不应直接替代正式字段名。
3. 当前 GUI 运行主线依赖 snapshot；不要把它理解为完全脱离 INP / Results 主线的独立求解器。
4. 3D 预览可能退回 placeholder，但这不自动表示结果不可读。

### 14. 常见错误与排查

| 现象 | 常见原因 | 处理方法 |
| --- | --- | --- |
| 某个按钮点开后只显示“Planned”或占位提示 | 该功能仍是 placeholder | 退回正式主线命令，不把该入口当作当前版本可执行工作流 |
| 结果已经打开，但视图区仍是占位提示 | 当前环境未创建 PyVista 预览控件 | 先确认结果树、ResultsBrowser、probe 和导出是否正常，再判断是否只是预览退化 |
| 模型能打开，但某些对象不能编辑 | 该对象类型未纳入正式编辑主线 | 优先编辑材料、截面、边界、载荷、步骤、输出请求和实例变换等正式对象 |
| GUI 里看到某个字段族名，就以为结果文件里一定有该字段 | GUI 展示层使用了字段族与别名 | 以正式字段列表和结果浏览器中的实际字段为准 |

### 15. 相关主题

- 第 3 章 快速开始
- 第 4 章 定义层与编译层
- 第 5 章 输入文件参考
- 后续章节中的“结果层”
- 后续章节中的“作业、快照与派生算例”

## 第 7 章 Python API / Facade / Probe
本章只讲产品壳层与结果消费入口，不再代讲编译层、kernel、procedures 或 solver 内部对象。读者看完本章后，应能回答两个问题：普通脚本应该从哪里进入主线，结果浏览应停留在哪一层。

### 1. 这一章为什么存在

pyFEM 需要一层面向用户和脚本的稳定入口，用来屏蔽底层编译、过程执行和结果协议细节。这一层的正式职责是：

1. 把“读模、运行、打开结果、导出结果”收口为少量稳定 API。
2. 把 `ResultsFacade / Query / Probe` 收口为结果消费入口。
3. 让普通用户不必直接依赖 `Compiler`、`ProcedureRuntime`、`Assembler` 或 `RuntimeState`。

因此，本章属于产品壳层 / 消费层，不属于底层结构章。

### 2. 产品壳层对象家族地图

下表先给出本章涉及的对象家族地图。

| 家族 | 所属层 | 解决的问题 | 核心对象 | 与相邻对象关系 |
| --- | --- | --- | --- | --- |
| 会话入口家族 | 产品壳层 / 消费层 | 为脚本提供统一入口 | `PyFEMSession` | 上游无；下游调度 job、编译、结果读取与导出 |
| 结果门面家族 | 结果层消费侧 | 为用户提供 reader-only 结果入口 | `ResultsFacade` | 上游消费 `ResultsReader`；下游派生 query / probe |
| 结果查询家族 | 结果层消费侧 | 做批量筛选与概览 | `ResultsQueryService` | 由 facade 创建，消费 `ResultsDatabase` |
| 结果探针家族 | 结果层消费侧 | 抽取可直接画图或导出的序列 | `ResultsProbeService` | 由 facade 创建，消费字段或历史量对象 |

这里要特别注意：`JobManager` 属于产品壳层，但它的统一执行流程放在第 13 章讲。本章只把它作为 `PyFEMSession` 背后的主线调度器提及，不重复展开。

### 3. `PyFEMSession`

- 所属层：产品壳层 / 消费层
- 家族：会话入口家族
- 职责：作为 Python API 的统一会话入口，收口读模、运行、打开结果和导出结果
- 核心字段：未显式传入依赖时，当前会准备 `registry`、`plugin_discovery`、`job_manager`
- 主要方法状态：属于典型服务类
- 上下游关系：上游面向用户脚本；下游调度 importer、`JobManager`、reader、exporter
- 典型误解：它不是 solver 对象，也不是 `ResultsDatabase`

下表只列当前建议写成正式公开边界的方法。

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 | 常见误解 |
| --- | --- | --- | --- | --- | --- | --- |
| `load_model_from_file()` | `input_path`，可选 `model_name`、`importer_key` | `ModelDB` | 只读模，不执行 | 想先读模再检查时 | 普通用户、高级脚本 | 它不负责编译和求解 |
| `run_input_file()` | 输入文件路径与可选步骤、结果、导出参数 | `JobExecutionReport` | 从输入文件走完整主线 | 已有输入文件时 | 普通用户首选、GUI、脚本 | 它不仅仅是“调用求解器” |
| `run_model()` | `ModelDB` 与可选步骤、结果、导出参数 | `JobExecutionReport` | 对内存模型走完整主线 | 程序化建模时 | 高级脚本、测试 | 默认路径推导弱于 `run_input_file()` |
| `open_results()` | `results_path`，可选 `backend_key` | `ResultsFacade` | 打开结果门面 | 结果已经存在时 | 普通用户、高级脚本、GUI | 返回的不是 `ResultsDatabase` 本体 |
| `export_results()` | `input_path`、`results_path`、`export_path` 与可选格式/步骤 | `Path` | 重新加载模型与结果后执行导出 | 已有结果，后补导出时 | 普通用户、高级脚本 | 不是直接从 JSON 任意转格式 |
| `discover_plugin_manifests()` | 零个或多个搜索根 | manifest 序列 | 只发现插件清单 | 检查插件目录时 | 维护者、插件开发者 | 发现不等于已注册 |
| `register_discovered_plugins()` | 零个或多个搜索根 | manifest 序列 | 发现并注册插件 | 需要把插件接入当前会话时 | 维护者、插件开发者 | 它不替代插件自身实现校验 |

对普通脚本，最稳妥的推荐路径仍然是：

```text
PyFEMSession
    -> run_input_file() / run_model()
    -> JobExecutionReport
    -> open_results()
    -> ResultsFacade
```

### 4. `ResultsFacade`

- 所属层：结果层消费侧
- 家族：结果门面家族
- 职责：把 reader-only 结果浏览收口成统一入口
- 核心字段：当前以 reader 引用和能力边界为主
- 主要方法状态：属于门面类
- 上下游关系：上游来自 `open_results()`；下游派生 query / probe
- 典型误解：它不是结果数据库对象

下表只列最常用的正式入口。

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 | 常见误解 |
| --- | --- | --- | --- | --- | --- | --- |
| `session()` | 无 | `ResultsSession` | 读取会话元数据 | 打开结果后 | 脚本、GUI | 不是数据库本体 |
| `capabilities()` | 无 | 能力描述对象 | 查看 reader 能力边界 | 诊断或适配时 | 高级脚本、GUI | 不是 writer 能力 |
| `list_steps()` | 无 | `tuple[str, ...]` | 列出步骤名 | 浏览前 | 脚本、GUI | 不做筛选 |
| `step()` | `step_name` | `ResultStep` | 读取单步对象 | 已知步名时 | 脚本、GUI | 不是 query 的别名 |
| `frame()` | `step_name`、`frame_id` | `ResultFrame` | 读取单帧 | 已知帧号时 | 脚本、GUI | 不是 probe |
| `field()` | `step_name`、`frame_id`、`field_name` | `ResultField` | 读取单字段对象 | 想直接取字段时 | 脚本、GUI | 不会自动拆成曲线 |
| `history()` | `step_name`、`history_name` | `ResultHistorySeries` | 读取单个历史量对象 | 想看 history 本体时 | 脚本、GUI | 与 probe 的 `history()` 返回不同 |
| `summary()` | `step_name`、`summary_name` | `ResultSummary` | 读取步骤摘要 | 查看统计信息时 | 脚本、GUI | 不是普通日志 |
| `query()` | 无 | `ResultsQueryService` | 下钻到查询服务 | 需要筛选和概览时 | 高级脚本、GUI | 不是结果数据库 |
| `probe()` | 无 | `ResultsProbeService` | 下钻到探针服务 | 需要序列抽取时 | 普通用户、高级脚本、GUI | 不是任意字段都能自动猜测目标和分量 |

### 5. `ResultsQueryService`

- 所属层：结果层消费侧
- 家族：结果查询家族
- 职责：按步骤、帧、字段、历史量和摘要做筛选与概览
- 核心字段：当前以服务边界为主
- 主要方法状态：属于服务类
- 上下游关系：由 `ResultsFacade.query()` 创建
- 典型误解：它返回的是结果对象或概览，不是 probe 序列

当前公开方法至少包括：

- `steps()`、`step()`
- `frames(...)`
- `field()`、`field_overview()`、`field_overviews()`
- `history()`、`histories(...)`
- `summary()`、`summaries(...)`

如果用户要回答“有哪些步骤、有哪些帧、哪些字段满足条件”，应优先用 query，而不是直接手动遍历 `ResultsDatabase`。

### 6. `ResultsProbeService`

- 所属层：结果层消费侧
- 家族：结果探针家族
- 职责：把结果对象转成可直接消费的探针序列，并支持 CSV 导出
- 核心字段：当前以服务边界为主
- 主要方法状态：属于服务类
- 上下游关系：由 `ResultsFacade.probe()` 创建
- 典型误解：probe 不是 `ResultField` 的同义词

当前公开方法至少包括：

- `history()`、`paired_history()`
- `field_component()`
- `node_component()`
- `element_component()`
- `integration_point_component()`
- `averaged_node_component()`
- `export_csv()`

这里最重要的使用分工是：

- `ResultsFacade.field()` 读取字段对象本体。
- `ResultsQueryService.field_overviews()` 读取字段概览。
- `ResultsProbeService.node_component()` / `field_component()` 把字段投影成一条序列。

### 7. 对象创建、引用与消费短例

#### 7.1 从输入文件直接运行

```python
from pyfem import PyFEMSession

session = PyFEMSession()
report = session.run_input_file(
    "tests/data/beam/b21_cantilever.inp",
    step_name="step-1",
    results_backend="json",
)
```

这条链路说明：

1. 普通用户入口应优先停在 `PyFEMSession`。
2. `run_input_file()` 内部会继续调度 job、编译、过程执行和结果写出。
3. 返回值是 `JobExecutionReport`，不是 `CompiledModel`。

#### 7.2 打开结果并读取字段

```python
results = session.open_results(report.results_path)
u = results.field("step-1", 0, "U")
```

这条链路说明：

1. `open_results()` 打开的入口是 `ResultsFacade`。
2. `field()` 返回的是正式 `ResultField`。
3. 结果消费不需要直接接触 solver 内部状态。

#### 7.3 下钻 probe 并导出 CSV

```python
probe = results.probe()
series = probe.node_component("step-1", "part-1.n2", "UY")
probe.export_csv(series, "outputs/uy_probe.csv")
```

这条链路说明：

1. probe 服务消费的是结果对象，不是 `RuntimeState`。
2. `node_component()` 适合取节点量分量序列。
3. CSV 导出属于结果消费动作，不属于结果层本体定义。

### 8. 边界对照表

下表用于澄清本章最容易混淆的相邻对象。

| 对象或边界对 | 所属层 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- | --- |
| `PyFEMSession` vs `JobManager` | 产品壳层 | 一个是统一会话入口，一个是内部作业调度器 | 不是同一层 API 的不同名字 | 普通用户通常只需要前者 |
| `PyFEMSession` vs `Compiler` | 产品壳层 / 编译层 | 一个是用户入口，一个是底层编译服务 | 不是“高低级版本的同一接口” | 两者都可能出现在脚本里 |
| `ResultsFacade` vs `ResultsDatabase` | 结果消费侧 / 结果层 | 一个是门面，一个是结果本体 | 不是 reader 与 payload 的同义词 | facade 经常被误写成数据库 |
| `ResultsQueryService` vs `ResultsProbeService` | 结果消费侧 | 一个做筛选与概览，一个做序列抽取 | 不是相同服务的不同别名 | 都由 facade 派生 |
| 产品壳层 vs solver | 产品壳层 / solver | 一个组织入口，一个执行离散求解 | 不是同一调用栈里的同层对象 | API 最终会触发 solver，但不应暴露其内部合同 |

### 9. 本章主线图

```text
PyFEMSession
    -> run_input_file() / run_model()
    -> JobExecutionReport
    -> open_results()
    -> ResultsFacade
    -> ResultsQueryService / ResultsProbeService
```

---

## 第 8 章 建模对象说明
本章说明 pyFEM 当前定义层对象的职责、关键字段、引用关系与校验规则。这里讲的是“模型怎样被描述”，不是“求解器怎样实现”。

### 1. 功能概述

pyFEM 当前采用以 `ModelDB` 为中心的定义层模型。部件、装配、材料、截面、边界条件、载荷、输出请求、步骤和作业都在这一层组织。

编译之前，用户看到的是一组“定义对象”。
编译之后，求解器使用的是“运行时对象”。
本章只关注前者。

### 2. 顶层对象 `ModelDB`

#### 2.1 对象职责

`ModelDB` 是问题定义的唯一汇总容器。当前可确认的主要字段包括：

- `parts`
- `assembly`
- `materials`
- `sections`
- `boundaries`
- `nodal_loads`
- `distributed_loads`
- `interactions`
- `output_requests`
- `steps`
- `raw_keyword_blocks`
- `job`

这意味着：

- 网格和几何组织在 `parts` 与 `assembly`
- 物理属性组织在 `materials` 与 `sections`
- 工况组织在 `boundaries`、`nodal_loads`、`distributed_loads`、`output_requests`
- 求解流程组织在 `steps` 与 `job`

#### 2.2 编译作用域

`ModelDB.iter_compilation_scopes()` 会决定编译时实际遍历的作用域：

- 如果没有装配实例，作用域通常就是各个 `Part`
- 如果存在 `Assembly` 和 `PartInstance`，作用域就切换为各个实例

这条规则非常重要，因为后续很多名称解析、结果命名和目标查找都依赖“作用域”而不是只依赖“部件”。

#### 2.3 `ModelDB` 的公开方法与调用时机

`ModelDB` 不是一个“把对象全塞进去的大字典”。它已经有明确的容器方法和校验入口。读者真正需要掌握的是：对象应通过哪些入口进入模型，以及这些入口何时会参与后续编译主线。

| 方法 | 当前作用 | 典型输入 | 什么时候用 | 谁会继续消费 |
| --- | --- | --- | --- | --- |
| `add_part()` | 注册部件 | `Part` | 建立几何与网格时 | `iter_compilation_scopes()`、`validate()` |
| `get_part()` | 读取部件 | `part_name` | 复用部件或检查装配前 | 脚本、装配构造代码 |
| `set_assembly()` | 绑定装配 | `Assembly` | 进入实例化建模时 | 作用域解析、编译层 |
| `add_material()` / `add_section()` | 注册物理定义 | `MaterialDef`、`SectionDef` | 材料和截面准备完成后 | provider 绑定、编译映射 |
| `add_boundary()` / `add_nodal_load()` / `add_distributed_load()` | 注册工况定义 | 对应 `Def` 对象 | 步骤引用之前 | `StepDef`、`validate()` |
| `add_output_request()` | 注册输出请求 | `OutputRequest` | 步骤引用之前 | 输出规划器 |
| `add_step()` | 注册步骤 | `StepDef` | 把工况串成分析步骤时 | procedure provider |
| `set_job()` | 注册作业顺序 | `JobDef` | 多步执行顺序确定后 | 作业执行链 |
| `iter_compilation_scopes()` | 枚举编译作用域 | 无 | 编译前、调试作用域时 | `Compiler` |
| `resolve_compilation_scope()` | 定位单个作用域 | `scope_name` | 查找实例或部件作用域时 | 编译桥接逻辑 |
| `validate()` | 执行定义层校验 | 无 | 编译前、自检时 | 用户、导入器、编译服务 |

很多初学者会直接手改 `model.parts[...]` 或 `model.steps[...]`。源码虽然允许读这些字段，但帮助文档更推荐走 `add_*` / `set_*` 方法，因为这样更符合正式主线，也更容易和后续校验逻辑对齐。

#### 2.4 最小构造示例：把定义对象装入 `ModelDB`

下面的例子只做一件事：让读者看到 `ModelDB` 里的对象不是平铺词条，而是一条连续定义链。

```python
mesh = Mesh()
mesh.add_node(NodeRecord(name="n1", coordinates=(0.0, 0.0)))
mesh.add_node(NodeRecord(name="n2", coordinates=(2.0, 0.0)))
mesh.add_element(ElementRecord(name="beam-1", type_key="B21", node_names=("n1", "n2")))
mesh.add_node_set("root", ("n1",))
mesh.add_node_set("tip", ("n2",))
mesh.add_element_set("beam-set", ("beam-1",))

part = Part(name="beam-part", mesh=mesh)

model = ModelDB(name="beam-demo")
model.add_part(part)
model.add_material(
    MaterialDef(
        name="mat-steel",
        material_type="linear_elastic",
        parameters={"young_modulus": 2.1e11, "poisson_ratio": 0.3, "density": 7850.0},
    )
)
model.add_section(
    SectionDef(
        name="sec-beam",
        section_type="beam",
        material_name="mat-steel",
        region_name="beam-set",
        scope_name="beam-part",
        parameters={"area": 3.0e-4, "moment_inertia_z": 8.1e-9},
    )
)
```

这段代码完成了三层绑定：

1. `NodeRecord` 和 `ElementRecord` 先进入 `Mesh`。
2. `Mesh` 再进入 `Part`，部件成为模型的几何容器。
3. `SectionDef` 通过 `region_name="beam-set"` 和 `scope_name="beam-part"` 命中单元区域，把物理定义挂到部件作用域上。

### 3. 网格与装配对象

#### 3.1 `Part`

`Part` 表示一个可复用的部件，内部持有 `Mesh`。

如果模型没有显式装配实例，部件通常既是几何容器，也是编译作用域。

#### 3.2 `Mesh`

`Mesh` 当前可确认的主要字段包括：

- `nodes`
- `elements`
- `node_sets`
- `element_sets`
- `surfaces`
- `orientations`

其中：

- `nodes` 保存节点记录
- `elements` 保存单元记录
- `node_sets` 与 `element_sets` 提供命名集合
- `surfaces` 提供面载荷和表面目标
- `orientations` 保存局部方向定义

#### 3.3 `NodeRecord`

`NodeRecord` 的核心字段是：

- `name`
- `coordinates`

当前校验规则要求同一网格内节点空间维数保持一致。

#### 3.4 `ElementRecord`

`ElementRecord` 的核心字段包括：

- `name`
- `type_key`
- `node_names`
- `section_name=None`
- `material_name=None`
- `region_name=None`
- `orientation_name=None`
- `properties={}`

这里有几个容易混淆的点：

- `type_key` 指向单元类型，例如 `C3D8`、`CPS4`、`B21`
- `section_name` 和 `material_name` 可以直接挂在单元上
- `region_name` 则允许单元通过区域名称与截面、材料等定义建立间接关联
- `orientation_name` 用于引用局部方向定义

#### 3.5 `SurfaceFacet` 与 `Surface`

`Surface` 由若干 `SurfaceFacet` 组成，用于表达“某个命名表面包含哪些单元面”。

这类对象是表面载荷和表面目标选择的基础。

#### 3.6 `Orientation`

`Orientation` 当前可确认的主要字段包括：

- `name`
- `system="rectangular"`
- `axis_1`
- `axis_2`
- `parameters`

当前校验会检查：

- 轴向量长度必须大于零
- 维数必须与模型空间维数一致
- 两个轴向量必须线性无关

因此，`Orientation` 不是普通标记对象，而是带有严格几何约束的正式定义对象。

#### 3.7 `RigidTransform`

`RigidTransform` 表示实例级刚体变换，当前会校验：

- 维数必须是 2D 或 3D
- 旋转矩阵应保持正交
- 旋转矩阵应保持右手系

这说明装配实例的空间变换已经进入正式数据模型范围，而不是只在 GUI 层做显示。

#### 3.8 `PartInstance`

`PartInstance` 当前可确认字段包括：

- `name`
- `part_name`
- `transform`

一个实例总是引用一个现有部件，再通过实例名和刚体变换形成实际求解作用域。

#### 3.9 `Assembly`

`Assembly` 当前不仅保存 `instances`，还保存实例级别的别名索引：

- `instance_node_sets`
- `instance_element_sets`
- `instance_surfaces`

这说明装配层并不是“只有实例列表”，而是已经承担实例级目标解析职责。

#### 3.10 网格记录家族的数据长相

下表只说明网格记录对象的字段形态，不讨论编译去向。

| 对象 | 关键字段 | 数据格式 | 典型值 | 读者最该记住的点 |
| --- | --- | --- | --- | --- |
| `NodeRecord` | `name`、`coordinates` | `str`、`tuple[float, ...]` | `("n2", (2.0, 0.0))` | 节点记录只描述名字和坐标，不描述自由度 |
| `ElementRecord` | `name`、`type_key`、`node_names`、`section_name`、`material_name`、`region_name`、`orientation_name`、`properties` | `str`、`tuple[str, ...]`、可空 `str`、`dict[str, Any]` | `type_key="CPS4"`、`node_names=("n1","n2","n3","n4")` | 它是定义记录，不是单元刚度对象 |
| `SurfaceFacet` | `element_name`、`local_face` | `str`、`str` | `("block-1", "S3")` | 面片总是指向已有单元和该单元局部面 |
| `Surface` | `name`、`facets` | `str`、`tuple[SurfaceFacet, ...]` | `name="top"` | 分布载荷和表面目标先命中它，再命中单元面 |
| `Orientation` | `name`、`system`、`axis_1`、`axis_2`、`parameters` | `str`、`tuple[float, ...]`、`dict[str, Any]` | `axis_1=(1,0,0)` | 当前会校验轴长度、维数和线性无关性 |
| `RigidTransform` | `rotation`、`translation` | `tuple[tuple[float, ...], ...]`、`tuple[float, ...]` | 二维平移 `(3.0, 0.0)` | 它不是显示偏移，而是实例级正式几何变换 |

对 `ElementRecord`，最容易出错的是把字段都当成“随便写个名字就行”。实际上：

- `node_names` 必须命中当前 `Mesh.nodes`。
- `orientation_name` 必须命中 `Mesh.orientations`。
- `section_name` / `material_name` / `region_name` 不是同义字段，它们对应不同的引用路径。

#### 3.11 `Part`、`Mesh`、`Assembly` 的公开方法

定义层里最常被直接调用的不是 dataclass 构造，而是这些容器方法。

| 类 | 常用方法 | 当前语义 |
| --- | --- | --- |
| `Mesh` | `add_node()`、`add_element()`、`add_node_set()`、`add_element_set()`、`add_surface()`、`add_orientation()` | 把记录对象和命名集合装入网格 |
| `Mesh` | `get_node()`、`get_element()`、`has_node_set()`、`has_element_set()`、`has_surface()` | 读取单个记录或查询命名对象是否存在 |
| `Mesh` | `spatial_dimension()`、`validate()` | 计算网格维数并执行一致性校验 |
| `Part` | `add_node()`、`add_element()` 等 | 这些方法会直接转发给内部 `Mesh` |
| `Part` | `get_node()`、`get_element()`、`validate()`、`spatial_dimension()` | 以部件视角访问网格内容 |
| `Assembly` | `add_instance()` | 注册一个 `PartInstance` |
| `Assembly` | `add_instance_node_set()`、`add_instance_element_set()`、`add_instance_surface()` | 为实例建立作用域内别名 |
| `Assembly` | `node_sets_for_instance()`、`element_sets_for_instance()`、`surfaces_for_instance()` | 查询某个实例名下可用的命名对象 |

因此，`Part` 更像“带名字的部件容器”，`Mesh` 是“真正保存记录和集合的地方”，`Assembly` 则是“把部件放入实例作用域并给目标查找加一层名字空间”。

#### 3.12 构造示例：`NodeRecord -> Mesh -> Part`

```python
mesh = Mesh()
mesh.add_node(NodeRecord(name="n1", coordinates=(0.0, 0.0)))
mesh.add_node(NodeRecord(name="n2", coordinates=(1.0, 0.0)))
mesh.add_node(NodeRecord(name="n3", coordinates=(1.0, 1.0)))
mesh.add_node(NodeRecord(name="n4", coordinates=(0.0, 1.0)))
mesh.add_orientation(Orientation(name="ori-1", axis_1=(1.0, 0.0, 0.0), axis_2=(0.0, 1.0, 0.0)))
mesh.add_element(
    ElementRecord(
        name="plate-1",
        type_key="CPS4",
        node_names=("n1", "n2", "n3", "n4"),
        orientation_name="ori-1",
    )
)
mesh.add_node_set("left", ("n1", "n4"))
mesh.add_element_set("plate-set", ("plate-1",))

part = Part(name="part-1", mesh=mesh)
```

这段代码里有两个很实用的阅读方法：

1. 看 `ElementRecord.node_names`，就知道单元连接关系的数据长相。
2. 看 `mesh.add_element_set("plate-set", ("plate-1",))`，就知道后续 `SectionDef.region_name` 要命中的字符串是什么。

#### 3.13 引用示例：`ElementRecord` 如何命中截面、材料和方向

下面两种写法都属于当前正式定义主线，但含义不同。

```python
# 直接绑定
mesh.add_element(
    ElementRecord(
        name="beam-1",
        type_key="B21",
        node_names=("n1", "n2"),
        section_name="sec-beam",
        material_name="mat-steel",
    )
)

# 区域绑定
mesh.add_element(ElementRecord(name="beam-2", type_key="B21", node_names=("n2", "n3"), region_name="beam-set"))
model.add_section(
    SectionDef(
        name="sec-beam",
        section_type="beam",
        material_name="mat-steel",
        region_name="beam-set",
        scope_name="beam-part",
        parameters={"area": 3.0e-4, "moment_inertia_z": 8.1e-9},
    )
)
```

区别在于：

- 直接绑定时，`ElementRecord` 自己就写明截面名和材料名。
- 区域绑定时，`ElementRecord` 只声明自己属于哪个区域，真正的截面和材料由 `SectionDef.region_name + scope_name` 来命中。

如果再叠加局部方向，则 `orientation_name` 会去 `Mesh.orientations` 里找同名 `Orientation`。这条引用链在编译前就会由 `validate()` 检查。

#### 3.14 装配示例：`PartInstance` 与 `Assembly`

```python
assembly = Assembly(name="assembly-1")
assembly.add_instance(PartInstance(name="left", part_name="beam-part"))
assembly.add_instance(
    PartInstance(
        name="right",
        part_name="beam-part",
        transform=RigidTransform(translation=(3.0, 0.0)),
    )
)
assembly.add_instance_node_set("left", "root", ("n1",))
assembly.add_instance_node_set("right", "root", ("n1",))

model.set_assembly(assembly)
```

这时读者需要换一个思路：

- `beam-part` 还是定义层里的部件名。
- `left` 和 `right` 已经是后续编译、载荷命中和结果命名所使用的实例作用域名。
- 同名集合 `root` 可以在不同实例下重复存在，真正区分它们的是实例作用域。

### 4. 物理与工况定义对象

#### 4.1 `MaterialDef`

`MaterialDef` 的核心字段是：

- `name`
- `material_type`
- `parameters`

当前正式主线可确认的材料类型包括：

- `linear_elastic`
- `elastic_isotropic`
- `j2_plastic`
- `j2_plasticity`

对内置线弹性材料，可以确认的参数别名包括：

- 杨氏模量：`young_modulus`、`elastic_modulus`、`e`
- 泊松比：`poisson_ratio`、`nu`
- 可选密度：`density`

对内置 J2 塑性材料，可以确认的参数包括：

- 线弹性参数同上
- 必需屈服应力：`yield_stress`
- 可选硬化模量：`hardening_modulus`
- 可选密度：`density`
- 可选切线模式：`tangent_mode`

#### 4.2 `SectionDef`

`SectionDef` 的核心字段是：

- `name`
- `section_type`
- `material_name=None`
- `region_name=None`
- `scope_name=None`
- `parameters`

当前正式主线可确认的截面类型包括：

- `solid`
- `plane_stress`
- `plane_strain`
- `beam`

内置截面的主要参数约束如下：

- `solid` 不要求额外几何参数，重点是材料绑定
- `plane_stress` 和 `plane_strain` 可使用 `thickness`，默认值为 `1.0`
- `beam` 需要 `area`
- `beam` 需要 `moment_inertia_z` 或别名 `iz`
- `beam` 可选 `shear_factor`，默认值为 `1.0`

此外，单元类型与截面类型之间存在正式兼容关系：

- `C3D8` 要求实体类截面
- `CPS4` 要求平面类截面
- `B21` 要求梁截面

文档写作时应把这些兼容关系视为正式限制，而不是经验建议。

#### 4.3 `BoundaryDef`

`BoundaryDef` 的核心字段包括：

- `name`
- `target_name`
- `dof_values`
- `target_type="node_set"`
- `scope_name=None`
- `boundary_type="displacement"`
- `parameters`

当前内置正式约束类型是 `displacement`。因此：

- 边界条件的正式主线是位移型约束
- `dof_values` 用于给出自由度值
- 目标默认按节点集解释

#### 4.4 `NodalLoadDef`

`NodalLoadDef` 的核心字段包括：

- `name`
- `target_name`
- `components`
- `target_type="node_set"`
- `scope_name=None`
- `parameters`

当前正式主线中，节点载荷目标类型应理解为：

- `node`
- `node_set`

载荷分量既可以直接写自由度名，也可以写常见力学别名。当前代码中可确认的映射包括：

- `FX`、`FY`、`FZ`
- `MX`、`MY`、`MZ`

这些别名最终会映射到对应自由度方向。

如果 `parameters` 中给出了 `scale`，装配时会应用该缩放因子。

#### 4.5 `DistributedLoadDef`

`DistributedLoadDef` 的核心字段包括：

- `name`
- `target_name`
- `load_type`
- `components`
- `target_type="surface"`
- `scope_name=None`
- `parameters`

从定义对象角度看，分布载荷已经是正式对象；但从求解主线角度看，它的实际支持范围比对象定义本身更窄。

当前可以确认：

- 装配器只接受 `surface` 目标
- `parameters["scale"]` 会作为缩放因子参与装配
- `P` 和 `pressure` 会归一化到 `pressure`
- `follower`、`follower_pressure`、`follower-pressure` 会归一化到 `follower_pressure`

但这并不等于所有载荷类型都已经有正式求解支持。对于当前帮助文档，应把“小变形主线下的表面压力载荷”作为最稳妥的正式说明口径，而不要把 GUI 中露出的所有选项都写成正式能力。

#### 4.6 `OutputRequest`

`OutputRequest` 的核心字段包括：

- `name`
- `variables`
- `target_type="model"`
- `target_name=None`
- `scope_name=None`
- `position="node"`
- `frequency=1`
- `parameters`

它表示“这个步骤希望输出哪些变量、面向什么目标、以什么位置和频率输出”。

需要特别注意：

- `variables` 不能为空
- `frequency` 必须大于零
- 位置、目标类型和变量类型之间存在正式匹配关系

当前正式支持的位置可确认包括：

- `NODE`
- `ELEMENT_CENTROID`
- `INTEGRATION_POINT`
- `ELEMENT_NODAL`
- `NODE_AVERAGED`
- `GLOBAL_HISTORY`

目标类型的正式校验范围可确认包括：

- `model`
- `node`
- `node_set`
- `element`
- `element_set`
- `surface`

但并不是所有目标类型都能和所有输出位置自由组合，具体匹配规则见本章第 6 节和第 9 章。

#### 4.7 `StepDef`

`StepDef` 的核心字段包括：

- `name`
- `procedure_type`
- `boundary_names=()`
- `nodal_load_names=()`
- `distributed_load_names=()`
- `output_request_names=()`
- `parameters`

它的职责是把“使用什么分析过程”与“调用哪些工况对象”绑定起来。

边界条件、载荷和输出请求本身不会自动生效。只有被步骤对象显式引用后，它们才进入该步骤的正式执行范围。

#### 4.8 `JobDef`

`JobDef` 的核心字段包括：

- `name`
- `step_names`
- `parameters`

作业对象负责组织步骤顺序。它说明 pyFEM 当前已经把“多步骤执行顺序”纳入正式模型层，而不是单纯依赖外部脚本逐步调用。

#### 4.9 关键定义对象的数据长相

下表把“物理定义对象”和“工况定义对象”的字段语义落到最小可用层面。

| 对象 | 关键字段 | 数据格式 | 典型值 | 当前正式边界 |
| --- | --- | --- | --- | --- |
| `MaterialDef` | `name`、`material_type`、`parameters` | `str`、`str`、`dict[str, Any]` | `material_type="linear_elastic"` | 当前正式内置主线是线弹性与 J2 塑性 |
| `SectionDef` | `name`、`section_type`、`material_name`、`region_name`、`scope_name`、`parameters` | `str`、可空 `str`、`dict[str, Any]` | `section_type="beam"` | 截面类型必须和单元类型兼容 |
| `BoundaryDef` | `target_name`、`target_type`、`scope_name`、`dof_values` | `str`、`dict[str, float]` | `dof_values={"UX": 0.0}` | 当前正式边界是位移型约束 |
| `NodalLoadDef` | `target_name`、`target_type`、`scope_name`、`components` | `str`、`dict[str, float]` | `components={"FY": -12.0}` | 当前正式目标是 `node` / `node_set` |
| `DistributedLoadDef` | `target_name`、`target_type`、`scope_name`、`load_type`、`components` | `str`、`dict[str, float]` | `load_type="pressure"` | 当前帮助文档应保守写为表面分布载荷对象，其中正式求解主线更窄 |
| `InteractionDef` | `name`、`interaction_type`、`scope_name`、`parameters` | `str`、可空 `str`、`dict[str, Any]` | `interaction_type="noop"` | 当前正式主线不外推为完整接触框架 |
| `OutputRequest` | `variables`、`target_type`、`target_name`、`scope_name`、`position`、`frequency` | `tuple[str, ...]`、`str`、`int` | `variables=("U","RF")` | 位置、目标和变量三者必须匹配 |
| `StepDef` | `procedure_type`、各类 `*_names`、`parameters` | `str`、`tuple[str, ...]`、`dict[str, Any]` | `procedure_type="modal"` | 步骤只引用对象，不复制对象本体 |
| `JobDef` | `step_names`、`parameters` | `tuple[str, ...]`、`dict[str, Any]` | `step_names=("step-static","step-modal")` | 作业顺序是正式模型的一部分 |

`parameters` 字段需要单独说明。它不是任意附加信息的容器，而是 provider 和 runtime 会继续读取的正式输入。例如：

- `MaterialDef.parameters` 会被材料 provider 解释为材料常数。
- `SectionDef.parameters` 会被截面 provider 解释为几何参数。
- `StepDef.parameters` 则会被 procedure provider 解释为模态数、时间步长、非线性控制参数等步骤参数。

#### 4.10 最小工况示例：`SectionDef`、`BoundaryDef`、`OutputRequest`、`StepDef`

```python
model.add_material(
    MaterialDef(
        name="mat-1",
        material_type="linear_elastic",
        parameters={"young_modulus": 1.0e6, "poisson_ratio": 0.3, "density": 4.0},
    )
)
model.add_section(
    SectionDef(
        name="sec-1",
        section_type="beam",
        material_name="mat-1",
        region_name="beam-set",
        scope_name="beam-part",
        parameters={"area": 0.03, "moment_inertia_z": 2.0e-4},
    )
)
model.add_boundary(BoundaryDef(name="bc-root", target_name="root", dof_values={"UX": 0.0, "UY": 0.0, "RZ": 0.0}))
model.add_nodal_load(NodalLoadDef(name="load-tip", target_name="tip", components={"FY": -12.0}))
model.add_output_request(OutputRequest(name="field-u", variables=("U", "RF"), target_type="model", position="NODE"))
model.add_step(
    StepDef(
        name="step-static",
        procedure_type="static_linear",
        boundary_names=("bc-root",),
        nodal_load_names=("load-tip",),
        output_request_names=("field-u",),
    )
)
model.set_job(JobDef(name="job-1", step_names=("step-static",)))
```

从这个例子里可以直接看出三条正式引用关系：

1. `SectionDef.material_name="mat-1"` 命中 `MaterialDef.name`。
2. `StepDef.boundary_names / nodal_load_names / output_request_names` 命中对应定义对象的 `name`。
3. `JobDef.step_names` 命中 `StepDef.name`，把步骤真正排成执行顺序。

### 5. 名称、引用与作用域规则

#### 5.1 目标查找的正式类型

`ModelDB` 当前会把以下目标类型视为正式可查找对象：

- `model`
- `node`
- `node_set`
- `element`
- `element_set`
- `surface`

这套目标类型贯穿边界条件、载荷和输出请求校验。

#### 5.2 作用域名称 `scope_name`

`scope_name` 的意义可以概括为：“这个定义属于哪个部件或哪个实例作用域”。

在无装配实例模型中，`scope_name` 往往对应部件名。
在有装配实例模型中，`scope_name` 更应理解为实例作用域名。

因此：

- 同名节点集如果分属不同实例，需要靠 `scope_name` 区分
- 截面、边界、载荷和输出请求是否能命中目标，都会受 `scope_name` 影响

#### 5.3 直接绑定与区域绑定

单元与材料、截面的关系有两条主线：

- 直接在 `ElementRecord` 上通过 `section_name`、`material_name` 绑定
- 通过 `region_name` 与 `SectionDef.region_name` 等区域定义建立间接绑定

帮助文档中需要同时说明这两条能力，但要提醒读者保持命名一致，否则模型校验阶段就会失败。

#### 5.4 名称引用表

下表专门说明字符串名字是如何在定义层命中正式对象的。

| 引用字段 | 所属对象 | 解析范围 | 命中对象 | 典型例子 |
| --- | --- | --- | --- | --- |
| `node_names[*]` | `ElementRecord` | 当前 `Mesh.nodes` | `NodeRecord` | `("n1","n2","n3","n4")` |
| `orientation_name` | `ElementRecord` | 当前 `Mesh.orientations` | `Orientation` | `"ori-1"` |
| `section_name` | `ElementRecord` | `ModelDB.sections` | `SectionDef` | `"sec-1"` |
| `material_name` | `ElementRecord` | `ModelDB.materials` | `MaterialDef` | `"mat-1"` |
| `region_name` | `ElementRecord` | 当前编译作用域中的区域名称 | `SectionDef.region_name` 所指的区域绑定 | `"plate-set"` |
| `material_name` | `SectionDef` | `ModelDB.materials` | `MaterialDef` | `"mat-1"` |
| `target_name + target_type + scope_name` | `BoundaryDef` | 由 `ModelDB.iter_target_scopes()` 决定 | 节点、节点集、单元、单元集或表面 | `("root", "node_set", "left")` |
| `target_name + target_type + scope_name` | `NodalLoadDef` / `DistributedLoadDef` | 同上 | 目标节点集或表面 | `("top", "surface", "part-1")` |
| `target_name + target_type + scope_name` | `OutputRequest` | 同上 | 输出筛选目标 | `("tip", "node_set", "right")` |
| `boundary_names[*]` 等 | `StepDef` | `ModelDB.boundaries` / `nodal_loads` / `distributed_loads` / `output_requests` | 对应定义对象 | `"bc-root"` |
| `step_names[*]` | `JobDef` | `ModelDB.steps` | `StepDef` | `"step-static"` |

读者真正需要掌握的是：pyFEM 当前大量使用“名字引用”，但这些名字并不是全局无条件生效。只要目标涉及集合、表面或实例，就必须把 `scope_name` 一起看。

#### 5.5 名称命中与运行时映射示例

下面这条链最适合作为阅读定义层的主线：

```text
ElementRecord.region_name = "beam-set"
    -> SectionDef.region_name = "beam-set"
    -> SectionDef.material_name = "mat-1"
    -> MaterialDef.name = "mat-1"
    -> Compiler.compile()
    -> SectionRuntime + MaterialRuntime + ElementRuntime
```

工况对象的命中过程可写成：

```text
BoundaryDef(target_name="root", target_type="node_set", scope_name="left")
    -> StepDef.boundary_names = ("bc-root",)
    -> ProcedureRuntime.run(...)
    -> Assembler.collect_constraints(...)
```

这类“名称如何命中正式对象并进入运行时映射”的过程，需要在帮助文档中明确写出。只列对象名而不说明名称解析规则，读者很难建立可复现的建模方法。

### 6. 输出请求的正式匹配规则

从输出请求规划器可以确认，输出位置、变量和目标类型存在明确匹配关系。

#### 6.1 节点类输出

节点位置与节点平均位置常见变量包括：

- `U`
- `U_MAG`
- `RF`
- `MODE_SHAPE`
- `S_AVG`
- `S_VM_AVG`
- `S_PRINCIPAL_AVG`

这类输出的正式目标类型主要是：

- `model`
- `node`
- `node_set`

#### 6.2 单元类输出

单元质心、积分点和单元节点位置常见变量包括：

- `S`
- `E`
- `SECTION`
- `S_IP`
- `E_IP`
- `S_VM_IP`
- `S_PRINCIPAL_IP`
- `S_REC`
- `E_REC`
- `S_VM_REC`
- `S_PRINCIPAL_REC`

这类输出的正式目标类型主要是：

- `model`
- `element`
- `element_set`

#### 6.3 全局历史输出

当前可确认的全局历史变量包括：

- `TIME`
- `FREQUENCY`

这类输出只能面向：

- `target_type="model"`

并且不应再给出 `target_name` 或 `scope_name`。

#### 6.4 关于默认输出

这里需要区分两层逻辑：

- 过程侧的输出规划器有自己的默认输出规则
- 导入器在读入输入文件时，也可能自动补入一组默认输出请求

二者不是同一个概念。

因此，帮助文档不应简单写成“pyFEM 总会默认输出某某变量”，而应写成：

- 如果使用特定导入器工作流，导入器可能自动补入默认请求
- 如果从 Python API 直接构造 `ModelDB`，则应主动检查步骤实际引用了哪些输出请求

#### 6.5 最小输出请求示例

下面三个例子分别对应节点场、单元场和全局历史量，基本覆盖了当前最常见的正式输出请求写法。

```python
# 节点位移与反力
OutputRequest(
    name="field-node",
    variables=("U", "RF"),
    target_type="model",
    position="NODE",
)

# 单元应力与应变
OutputRequest(
    name="field-element",
    variables=("S", "E"),
    target_type="element_set",
    target_name="plate-set",
    scope_name="part-1",
    position="ELEMENT_CENTROID",
)

# 全局时间历史
OutputRequest(
    name="history-time",
    variables=("TIME",),
    target_type="model",
    position="GLOBAL_HISTORY",
)
```

这三个例子各自回答了一个常见问题：

1. `U`、`RF` 这类节点量通常挂在 `NODE`。
2. `S`、`E` 这类单元量通常挂在 `ELEMENT_CENTROID`、`INTEGRATION_POINT` 或派生位置。
3. `TIME`、`FREQUENCY` 这类全局历史量不应再写 `target_name`。

### 7. 模型校验规则

`ModelDB.validate()` 当前会执行一组正式校验。帮助文档中至少应明确以下几类错误会在编译前被拦截：

- 模型至少要有一个部件
- 部件和网格本身必须有效
- 单元直接引用的截面和材料必须存在
- 装配实例引用的部件必须存在
- 实例变换维数必须匹配
- 截面引用的作用域、材料和区域必须存在
- 边界条件、节点载荷和分布载荷的目标必须存在
- 相互作用的作用域必须存在
- 输出请求如果不是面向 `model`，则必须给出有效目标
- 步骤必须给出 `procedure_type`
- 步骤引用的边界、载荷和输出请求名字必须存在
- 作业引用的步骤名字必须存在

### 8. 对象创建、引用与编译长例

下面的示例把“记录对象 -> 定义对象 -> 步骤引用 -> 编译产物”连成一个完整过程，可作为第 8 章的综合示例。

```python
mesh = Mesh()
mesh.add_node(NodeRecord(name="n1", coordinates=(0.0, 0.0)))
mesh.add_node(NodeRecord(name="n2", coordinates=(2.0, 0.0)))
mesh.add_element(ElementRecord(name="beam-1", type_key="B21", node_names=("n1", "n2"), region_name="beam-set"))
mesh.add_node_set("root", ("n1",))
mesh.add_node_set("tip", ("n2",))
mesh.add_element_set("beam-set", ("beam-1",))

model = ModelDB(name="beam-demo")
model.add_part(Part(name="beam-part", mesh=mesh))
model.add_material(
    MaterialDef(
        name="mat-1",
        material_type="linear_elastic",
        parameters={"young_modulus": 1.0e6, "poisson_ratio": 0.3, "density": 4.0},
    )
)
model.add_section(
    SectionDef(
        name="sec-1",
        section_type="beam",
        material_name="mat-1",
        region_name="beam-set",
        scope_name="beam-part",
        parameters={"area": 0.03, "moment_inertia_z": 2.0e-4},
    )
)
model.add_boundary(BoundaryDef(name="bc-root", target_name="root", dof_values={"UX": 0.0, "UY": 0.0, "RZ": 0.0}))
model.add_nodal_load(NodalLoadDef(name="load-tip", target_name="tip", components={"FY": -12.0}))
model.add_output_request(OutputRequest(name="field-u", variables=("U",), target_type="model", position="NODE"))
model.add_step(
    StepDef(
        name="step-static",
        procedure_type="static_linear",
        boundary_names=("bc-root",),
        nodal_load_names=("load-tip",),
        output_request_names=("field-u",),
    )
)
model.set_job(JobDef(name="job-1", step_names=("step-static",)))

compiled = Compiler().compile(model)
```

把这段代码拆开看，会更容易理解每个对象什么时候出现：

1. `NodeRecord`、`ElementRecord`、节点集和单元集先进入 `Mesh`。
2. `Part` 把 `Mesh` 变成模型中的正式部件。
3. `SectionDef` 通过 `region_name + scope_name` 命中 `beam-part` 作用域中的 `beam-set`。
4. `BoundaryDef` 和 `NodalLoadDef` 只有在被 `StepDef` 引用后才会成为该步骤的正式输入。
5. `Compiler.compile()` 之后，这些定义对象不会直接交给 solver，而会被翻译成编译产物、kernel runtime 和 procedure runtime。

最常见的误解主要有四个：

- `ElementRecord` 不是单元运行时对象，它不会自己算刚度。
- `BoundaryDef` 不是一创建就自动生效，它必须被步骤引用。
- `OutputRequest` 不是“结果文件设置面板”，它本身就是正式定义对象。
- `JobDef` 不是可有可无的装饰；一旦存在多步，步骤顺序就应写在这里。

### 9. 建模建议

- 优先把对象命名统一成“作用域名.对象名”的稳定风格，便于多实例模型追踪。
- 对 `OutputRequest`，不要依赖默认行为，显式写出变量、位置和目标会更稳妥。
- 对 `DistributedLoadDef`，把正式文档口径控制在当前求解主线可证实的范围内。
- 对 `Orientation` 和 `RigidTransform`，应视为正式模型对象，不要把它们写成 GUI 附属元数据。
- 对多步骤模型，尽量在 `JobDef.step_names` 中明确步骤顺序，而不是依赖脚本外部约定。

---

## 第 9 章 Procedures 层
本章把 `procedures` 作为独立体系讲清楚。它负责把一个 `StepDef` 组织成可执行的过程对象，并在步骤边界上管理状态推进、结果写出和对 solver 的调用。

### 1. 这一层为什么存在

`procedures` 层解决的是“一个分析步骤怎样执行”。它不负责局部物理细节，也不负责全局线性代数求解，而是承担以下正式职责：

1. 把 `StepDef` 翻译成 `ProcedureRuntime`。
2. 按过程类型组织边界、载荷、输出和求解主线。
3. 调用 solver 层完成离散系统装配与求解。
4. 在步骤开始和结束处处理状态继承、结果帧、历史序列和摘要。

因此，`procedures` 不是 `kernel` 的一部分，也不是 `solver` 的别名。它处在两者之上，负责“步骤级编排”。

### 2. Procedures 家族地图

下表先给出本层对象家族地图，帮助读者先建立层内分工，再进入具体对象。

| 家族 | 所属层 | 解决的问题 | 核心对象 | 与相邻对象关系 |
| --- | --- | --- | --- | --- |
| 抽象过程家族 | procedures 层 | 定义所有过程运行时对象的最小公共边界 | `ProcedureRuntime`、`ProcedureReport` | 由步骤定义和 provider 构造，并向 solver 与结果写出接口发起调用 |
| 步骤过程家族 | procedures 层 | 为基于步骤的正式过程提供通用状态与结果组织能力 | `StepProcedureRuntime` | 持有 `CompiledModel`、`DiscreteProblem` 和 `ResultsWriter` 所需的组织逻辑 |
| 内置过程家族 | procedures 层 | 提供当前正式主线中的静力、模态和隐式动力过程 | `StaticLinearProcedure`、`StaticNonlinearProcedure`、`ModalProcedure`、`ImplicitDynamicProcedure` | 读取 kernel runtime 提供的局部物理能力，并通过 solver 执行 |
| 过程 provider 家族 | procedures 层 | 把 `StepDef.procedure_type` 绑定到具体过程实现 | `StaticLinearProcedureProvider`、`StaticNonlinearProcedureProvider`、`ModalProcedureProvider`、`ImplicitDynamicProcedureProvider` | 负责连接编译结果与具体过程对象 |

### 3. `StepDef` 与 `ProcedureRuntime` 的区别

下表用于澄清最容易混淆的相邻对象。

| 对象 | 所属层 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- | --- |
| `StepDef` | 定义层 | 描述步骤类型、工况引用和参数的定义对象 | 不直接执行求解 | 看到 `procedure_type` 容易误以为它已经是运行时过程 |
| `ProcedureRuntime` | procedures 层 | 描述“这一过程怎样执行”的运行时对象 | 不是纯数据记录对象 | 它读取 `StepDef`，但两者不是同层对象 |

`StepDef` 的职责是“定义这个步骤要做什么”。`ProcedureRuntime` 的职责是“真正按这个步骤去执行什么”。这也是为什么第 9 章必须独立于第 4 章。

### 4. Procedure Provider 与步骤家族

`StepDef.procedure_type` 不会直接实例化 `ProcedureRuntime`。当前正式主线里，它会先命中 provider，再由 provider 构造具体过程对象。

下表只说明当前正式支持的过程键与 provider 绑定。

| `procedure_type` | provider | 典型运行时对象 | 当前正式说明 |
| --- | --- | --- | --- |
| `static` / `static_linear` | `StaticLinearProcedureProvider` | `StaticLinearProcedure` | 小变形线性静力主线 |
| `modal` | `ModalProcedureProvider` | `ModalProcedure` | 小变形线性模态主线 |
| `implicit_dynamic` / `dynamic` | `ImplicitDynamicProcedureProvider` | `ImplicitDynamicProcedure` | Newmark 风格隐式动力主线 |
| `static_nonlinear` | `StaticNonlinearProcedureProvider` | `StaticNonlinearProcedure` | 增量迭代式非线性静力主线 |

provider 的正式角色包括：

1. 读取 `StepDef` 与 `CompiledModel`。
2. 选择合适的 `Assembler`、`DiscreteProblem` 和 `LinearAlgebraBackend`。
3. 创建具体 `ProcedureRuntime`。
4. 在必要时执行 fail-fast 校验。

其中，`StaticNonlinearProcedureProvider` 还会对 `nlgeom=True` 的元素、材料和载荷支持子集进行额外校验。因此，几何非线性边界不应被写成所有过程共享的公共承诺。

### 5. 各过程家族的共性与差异

下表只回答“这些过程家族分别组织哪类主线”，不展开全部算法细节。

| 过程对象 | 共性 | 关键差异 | 当前正式结果主线 |
| --- | --- | --- | --- |
| `StaticLinearProcedure` | 都基于 `StepProcedureRuntime` 组织结果和步骤边界 | 求解一次线性系统，不做增量迭代 | `U`、`RF`、`static_summary` |
| `ModalProcedure` | 都调用 solver 层而不是自己实现矩阵运算 | 走约化特征值求解主线 | `MODE_SHAPE`、`FREQUENCY` |
| `ImplicitDynamicProcedure` | 都读取 `StepDef` 中的边界、载荷和输出引用 | 按时间步推进，组织速度和加速度语义 | 时间步场结果、`dynamic_summary` |
| `StaticNonlinearProcedure` | 都通过 `DiscreteProblem` 调用 solver | 走增量迭代、cutback、状态继承主线 | 非线性历史量、`static_nonlinear_summary` |

### 6. 核心对象

#### 6.1 `ProcedureRuntime`

- 所属层：procedures 层
- 家族：抽象过程家族
- 职责：定义所有过程运行时对象的最小正式接口
- 核心字段：当前以抽象边界为主，具体字段由子类承担
- 主要方法状态：当前正式公开方法很少，属于“以最小抽象接口为主”的 runtime 类
- 相邻对象关系：由 provider 创建，并在执行中调用 solver 与结果写出接口
- 典型误解：不要把它看成 `StepDef` 的别名

下表只列当前建议写成正式公开边界的方法。

| 方法 | 所属类 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 | 常见误解 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `get_name()` | `ProcedureRuntime` | 无 | `str` | 返回过程名 | 结果写出、日志描述时 | provider、外层执行服务 | 不是步骤定义对象的字段直读 |
| `get_procedure_type()` | `ProcedureRuntime` | 无 | `str` | 返回过程类型标识 | 区分步骤家族时 | provider、上层服务 | 不等同于输入文件原字符串原样保留 |
| `describe()` | `ProcedureRuntime` | 无 | 映射对象 | 返回过程摘要 | 结果摘要或调试输出时 | 上层执行服务 | 不是结果数据库对象 |
| `run(results_writer)` | `ProcedureRuntime` | `ResultsWriter` | `ProcedureReport` | 执行整个步骤主线 | 正式求解时 | 作业执行链、过程调度方 | 不是普通用户常直接手调的 API |

#### 6.2 `StepProcedureRuntime`

- 所属层：procedures 层
- 家族：步骤过程家族
- 职责：为步骤型过程提供统一的状态起点、结果会话和结果组织能力
- 核心字段：`definition`、`compiled_model`、`problem`、`backend`
- 主要方法状态：公开方法比抽象基类更多，属于“有明确运行主线的 runtime 类”
- 相邻对象关系：读取 `StepDef` 与 `CompiledModel`，并组织 `DiscreteProblem` 的调用
- 典型误解：它不是 solver 对象，也不是 `CompiledModel` 的一部分

除上述字段外，初始化阶段还会组织 `output_planner`、`raw_field_service`、`recovery_service`、`averaging_service`、`derived_field_service` 等结果相关服务。它们服务于过程执行，但不改变本层与结果层的边界。

下表只列当前可确认为正式公开边界的方法。

| 方法 | 所属类 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 | 常见误解 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `get_name()` | `StepProcedureRuntime` | 无 | `str` | 返回步骤运行时名 | 过程执行中 | 上层过程调度 | 不是直接读取 `StepDef.name` 的简写 |
| `get_procedure_type()` | `StepProcedureRuntime` | 无 | `str` | 返回当前过程类型 | 过程分派时 | 上层过程调度 | 不是 provider 名称 |
| `describe()` | `StepProcedureRuntime` | 无 | 映射对象 | 返回当前步骤的摘要描述 | 结果和日志组织时 | 上层执行服务 | 不等同于结果摘要对象 |
| `build_results_session()` | `StepProcedureRuntime` | 无 | `ResultsSession` | 创建本步骤结果会话 | 步骤开始时 | `run()` 内部 | 不是打开已有结果数据库 |
| `build_initial_state()` | `StepProcedureRuntime` | 无 | `RuntimeState` | 构造初始零状态 | 无前序步骤时 | `run()` 内部 | 不是从结果反读状态 |
| `build_step_start_state()` | `StepProcedureRuntime` | 无 | `RuntimeState` | 构造本步起点状态 | 步骤启动时 | `run()` 内部 | 不是总是零状态 |
| `get_state_transfer_channel()` | `StepProcedureRuntime` | 无 | `str \| None` | 声明是否参与跨步状态继承 | 多步步骤切换时 | 状态继承逻辑 | 返回 `None` 不代表过程无状态 |
| `resolve_inherited_state()` | `StepProcedureRuntime` | 无 | `RuntimeState \| None` | 从 `CompiledModel` 取回前序状态 | 存在继承通道时 | `run()` 内部 | 不代表所有过程都一定继承 |
| `restore_problem_state_from_previous_step()` | `StepProcedureRuntime` | `RuntimeState` | 无 | 把前序状态恢复到当前问题对象 | 跨步续算时 | `run()` 内部 | 不是 solver 自动完成 |
| `publish_problem_state_to_following_steps()` | `StepProcedureRuntime` | 无 | 无 | 把当前步骤状态发布给后续步骤 | 步骤完成时 | `run()` 内部 | 不是结果写出 |
| `build_frame()` | `StepProcedureRuntime` | 当前状态等 | `ResultFrame` | 生成结果帧 | 每个输出时刻 | `run()` 内部 | 不是 GUI 帧对象 |
| `build_global_history_series()` | `StepProcedureRuntime` | 执行历史 | 序列集合 | 生成全局历史量 | 步骤结束或追加时 | `run()` 内部 | 不等同于 field 输出 |
| `build_summary()` | `StepProcedureRuntime` | 执行统计 | `ResultSummary` | 生成步骤摘要 | 步骤完成后 | `run()` 内部 | 不是一般日志文本 |

#### 6.3 当前正式过程实现分节展开

下面不再只列“有哪些实现类”，而是分别说明每一种正式过程在主线里怎么工作。

##### 6.3.1 `StaticLinearProcedure`

`StaticLinearProcedure` 是当前正式小变形线性静力过程类。该类在 `StepDef.procedure_type` 为 `static` 或 `static_linear` 时，由 `StaticLinearProcedureProvider` 创建，并以 `StepDef`、`CompiledModel`、`DiscreteProblem`、`SciPyBackend` 为构造输入。

###### 数据字段

`definition`  
类型为 `StepDef`。  
该字段保存当前步骤定义，其中直接使用的内容包括 `boundary_names`、`nodal_load_names`、`distributed_load_names`、`output_request_names` 和 `parameters`。

`compiled_model`  
类型为 `CompiledModel`。  
该字段保存编译产物，过程对象通过它间接访问已编译的单元、材料、截面、约束和结果规划信息。

`problem`  
类型为 `DiscreteProblem`。  
该字段是静力线性求解的离散问题入口，负责矩阵装配、约束施加和状态提交。

`backend`  
类型为 `LinearAlgebraBackend`。  
当前正式内置实现由 provider 固定为 `SciPyBackend`。

`output_planner` 及结果辅助服务字段  
类型为内部服务对象。  
这些字段在 `StepProcedureRuntime.__post_init__()` 中创建，包括输出规划器、原始字段构造、恢复、平均和派生字段服务。当前帮助文档不将这些内部字段扩展为稳定用户合同。

###### 正式公开方法

`get_name()`  
返回 `str`。  
返回步骤名，当前实现直接取自 `definition.name`。

`get_procedure_type()`  
返回 `str`。  
返回步骤过程类型，当前实现直接取自 `definition.procedure_type`。

`describe()`  
返回 `Mapping[str, Any]`。  
返回过程摘要，其中包含步骤名、过程类型、边界条件名称、载荷名称、输出请求名称和步骤参数。

`build_results_session()`  
返回 `ResultsSession`。  
该方法根据模型名、作业名、步骤名、过程类型和步骤参数创建本步骤的结果会话对象。

`run(results_writer)`  
返回 `ProcedureReport`。  
这是当前类的正式执行入口。方法内部依次调用 `problem.begin_trial()`、`problem.assemble_tangent()`、`problem.assemble_external_load()`、`problem.apply_constraints()` 和 `backend.solve_linear_system()`，得到位移解后提交状态，并按输出请求写出 `ResultFrame`、`ResultSummary` 和可选的 `TIME` 历史量。

###### 当前可直接确认的行为

`run()` 只求解一次线性方程组，不执行增量控制，不执行迭代收敛判定。

求解完成后，当前实现会将 `trial_state.velocity` 和 `trial_state.acceleration` 写成与位移向量同长度的零向量，并将 `trial_state.time` 设为 `0.0`。

结果写出时，当前正式字段主线至少包括节点位移 `U` 和节点反力 `RF`。摘要对象名称固定为 `static_summary`，当前至少包含 `residual_norm`、`load_norm` 和 `displacement_norm` 三个数值项。

当输出规划器请求 `TIME` 历史量时，当前实现会写出轴类型为 `FRAME_ID`、轴值为 `(0,)`、数据值为 `(0.0,)` 的单点历史序列。

###### 当前边界

该类不执行非线性增量迭代，不应写成会自动切换到 `StaticNonlinearProcedure`。

该类会读取 `distributed_load_names`，但不同分布载荷是否能形成正式求解主线，仍受单元和求解实现的当前支持范围约束。

当前文档未将 `describe()` 返回映射的完整键集合冻结为稳定公开合同。

###### 最小示例

```python
step = StepDef(
    name="step-static",
    procedure_type="static_linear",
    boundary_names=("bc-root",),
    nodal_load_names=("load-tip",),
    output_request_names=("field-node", "history-time"),
)
```

##### 6.3.2 `ModalProcedure`

`ModalProcedure` 是当前正式小变形线性模态过程类。该类在 `StepDef.procedure_type` 为 `modal` 时创建，用于计算约束条件下的广义特征值和振型。

###### 数据字段

`definition`  
类型为 `StepDef`。  
该字段保存模态步定义，当前可直接确认会读取 `boundary_names`、`output_request_names` 和 `parameters["num_modes"]`。

`compiled_model`  
类型为 `CompiledModel`。  
该字段提供模态步所需的编译结果、自由度管理信息和结果规划上下文。

`problem`  
类型为 `DiscreteProblem`。  
该字段提供刚度矩阵、质量矩阵、约束缩减和向量扩展接口。

`backend`  
类型为 `LinearAlgebraBackend`。  
当前内置实现使用 `SciPyBackend` 求解广义特征值问题。

###### 正式公开方法

`get_name()`  
返回 `str`。  
返回步骤名。

`get_procedure_type()`  
返回 `str`。  
返回过程类型键，当前值为 `modal`。

`describe()`  
返回 `Mapping[str, Any]`。  
返回步骤摘要，包含模态步使用的边界条件、输出请求和参数字典。

`build_results_session()`  
返回 `ResultsSession`。  
创建本步骤结果会话对象。

`run(results_writer)`  
返回 `ProcedureReport`。  
这是当前类的正式执行入口。当前实现先装配切线矩阵和质量矩阵，再对两者施加相同的边界约束，随后调用 `backend.solve_generalized_eigenproblem()` 求解特征对，并把缩减空间中的模态向量扩展回全局自由度。

###### 当前可直接确认的行为

`run()` 会从 `definition.parameters` 中读取 `num_modes`，若未显式给出，则当前源码默认值为 `6`。

对每个模态，当前实现会生成一个 `ResultFrame`，其 `frame_kind` 为 `MODE`，`axis_kind` 为 `MODE_INDEX`，`axis_value` 为当前模态序号。字段名固定为 `MODE_SHAPE`。

模态向量扩展回全局自由度后，当前实现会按最大绝对值归一化。

当输出规划器请求历史量 `FREQUENCY` 时，当前实现会写出频率序列，并在 `paired_values` 中同步写出 `eigenvalue`。

###### 当前边界

该类不推进时间，不写出静力位移平衡解，不应与 `StaticLinearProcedure` 混写。

当前文档未将模态向量归一化策略扩展为用户可配置合同。当前可确认行为是按最大绝对值归一化。

当前文档未进一步展开 `ResultFrame.metadata` 中模态相关元数据的完整键表；当前源码可确认至少包含模态序号、频率和特征值。

###### 最小示例

```python
step = StepDef(
    name="step-modal",
    procedure_type="modal",
    boundary_names=("bc-root", "bc-guide"),
    output_request_names=("field-modal", "history-modal"),
    parameters={"num_modes": 3},
)
```

##### 6.3.3 `ImplicitDynamicProcedure`

`ImplicitDynamicProcedure` 是当前正式隐式动力过程类。该类由 `ImplicitDynamicProcedureProvider` 创建，当前实现采用 Newmark 参数格式组织时间离散。

###### 数据字段

`definition`  
类型为 `StepDef`。  
当前可直接确认会读取 `boundary_names`、`nodal_load_names`、`distributed_load_names`、`output_request_names` 和时间积分参数。

`compiled_model`  
类型为 `CompiledModel`。  
该字段提供已编译的运行时对象集合和结果写出上下文。

`problem`  
类型为 `DiscreteProblem`。  
该字段负责质量矩阵、阻尼矩阵、切线矩阵、外载、规定值处理和状态提交。

`backend`  
类型为 `LinearAlgebraBackend`。  
当前 provider 使用 `SciPyBackend` 求解每个时间步中的线性系统。

###### 正式公开方法

`get_name()`  
返回 `str`。  
返回步骤名。

`get_procedure_type()`  
返回 `str`。  
返回过程类型键，当前可为 `implicit_dynamic` 或其兼容入口 `dynamic`。

`describe()`  
返回 `Mapping[str, Any]`。  
返回步骤摘要。

`build_results_session()`  
返回 `ResultsSession`。  
创建隐式动力步骤的结果会话对象。

`build_initial_state()`  
返回 `RuntimeState`。  
根据 `initial_displacement`、`initial_velocity`、`initial_acceleration` 这组步骤参数构造零时刻状态。参数未给出时，相关向量保持为零向量。

`run(results_writer)`  
返回 `ProcedureReport`。  
这是当前类的正式执行入口。方法先解析积分参数，建立初始状态并施加约束对应的规定位移、速度和加速度；随后装配质量矩阵、阻尼矩阵、切线矩阵和初始外载，求解初始加速度；再按照时间步循环构造有效刚度与有效右端，逐步更新位移、速度、加速度和时间，并在满足输出规划时写出结果帧。

###### 当前可直接确认的行为

当前实现会解析 `time_step`、`num_steps` 或 `total_time`、`beta`、`gamma`，并将其整理为统一的 Newmark 参数对象。

初始加速度由约束修正后的质量矩阵和右端项求解得到。右端项为 `initial_load - damping @ velocity - tangent @ displacement`。

每个时间步内，当前实现都会更新 `displacement`、`velocity`、`acceleration` 和 `time`，并把受约束自由度重新写回规定值。

结果写出方面，当前正式字段主线可直接确认至少包括节点位移 `U`。摘要对象名称固定为 `dynamic_summary`，当前至少包含 `scheme`、`time_step`、`num_steps`、`times` 和 `displacement_norms`。

###### 当前边界

当前帮助文档未将速度场和加速度场写成稳定公开结果字段，不能直接扩写为当前正式输出。

当前帮助文档未进一步展开阻尼矩阵的完整来源合同。`assemble_damping()` 接口存在，但不同单元是否返回非零阻尼矩阵，应以当前源码实现为准。

当前文档未把完整时间积分参数集合冻结为稳定输入合同；除 `time_step`、`num_steps`、`total_time`、`beta`、`gamma` 与初始状态向量参数外，其余细节应以源码实现为准。

###### 最小示例

```python
step = StepDef(
    name="step-dynamic",
    procedure_type="implicit_dynamic",
    boundary_names=("bc-root", "bc-guide"),
    output_request_names=("field-dynamic", "history-dynamic"),
    parameters={
        "time_step": 0.0005,
        "total_time": 0.005,
        "beta": 0.25,
        "gamma": 0.5,
        "initial_displacement": {"part-1.n2.UX": 0.01},
    },
)
```

##### 6.3.4 `StaticNonlinearProcedure`

`StaticNonlinearProcedure` 是当前正式静力非线性过程类。该类用于在静力载荷作用下按增量和迭代方式求解非线性平衡路径。

###### 数据字段

`definition`  
类型为 `StepDef`。  
该字段保存静力非线性步骤定义，当前可直接确认会读取 `boundary_names`、`nodal_load_names`、`distributed_load_names`、`output_request_names` 和非线性控制参数。

`compiled_model`  
类型为 `CompiledModel`。  
该字段提供已编译的运行时对象集合以及步骤间状态快照入口。

`problem`  
类型为 `DiscreteProblem`。  
该字段提供切线装配、残量装配、约束施加、状态提交与回滚接口。

`backend`  
类型为 `LinearAlgebraBackend`。  
当前 provider 使用 `SciPyBackend` 求解每次迭代中的线性系统。

###### 正式公开方法

`get_name()`  
返回 `str`。  
返回步骤名。

`get_procedure_type()`  
返回 `str`。  
返回过程类型键 `static_nonlinear`。

`get_state_transfer_channel()`  
返回 `str | None`。  
当前实现返回 `"solid_mechanics_history"`。该值用于在多步计算中标识本步提交状态的继承通道。

`describe()`  
返回 `Mapping[str, Any]`。  
返回步骤摘要。

`build_results_session()`  
返回 `ResultsSession`。  
创建静力非线性步骤的结果会话对象。

`build_step_start_state()`  
返回 `RuntimeState`。  
该方法用于构造本步起始状态。当前实现优先尝试读取前一相关步骤已发布的提交状态；若不存在可继承状态，则创建零状态。若步骤参数中显式提供初始位移、速度、加速度，则按参数覆盖。

`restore_problem_state_from_previous_step()`  
返回 `None`。  
该方法在步骤执行前将可继承的提交状态装入当前 `DiscreteProblem`。

`publish_problem_state_to_following_steps()`  
返回 `None`。  
该方法在步骤成功完成后，将当前提交状态发布到 `CompiledModel` 的步骤状态快照结构中。

`run(results_writer)`  
返回 `ProcedureReport`。  
这是当前类的正式执行入口。方法先解析 `StaticNonlinearParameters`，再创建 `LineSearchController`，恢复前序状态，构造本步初始状态，随后按增量控制求解。每个增量内部会尝试求解目标载荷因子；若失败且允许 cutback，则减半当前增量重新尝试；若收敛，则写出结果帧、历史量和最终摘要。

###### 当前可直接确认的行为

当前实现会从步骤参数中解析 `max_increments`、`initial_increment`、`min_increment`、`max_iterations`、`residual_tolerance`、`displacement_tolerance`、`allow_cutback`、`line_search`、`nlgeom`。

结果历史量当前至少包括 `load_factor`、`residual_norm`、`displacement_norm`、`iteration_count`，当输出规划器请求 `TIME` 时还会补写 `TIME`。

摘要对象名称固定为 `static_nonlinear_summary`。当前源码可直接确认至少包含 `residual_norm`、`displacement_norm`、`iteration_count`、`converged_increment_count`、`cutback_count` 以及平均迭代次数等统计量。

多步状态继承方面，当前实现通过 `"solid_mechanics_history"` 通道与 `CompiledModel.step_state_snapshots` 对接。

###### 当前边界

当 `nlgeom=True` 时，provider 会先执行 fail-fast 校验。当前正式几何非线性支持范围收口到 `B21` 的 `corotational` 路线、`CPS4` 的 `total_lagrangian` 路线和 `C3D8` 的 `total_lagrangian` 路线，并进一步限制材料与截面组合。

当 `nlgeom=True` 时，当前 provider 会拒绝 `distributed load`、`surface pressure` 和 `follower pressure` 当前构形语义。

`line_search` 当前是受控入口，但源码中的 `LineSearchController.describe()` 仍将启用后的模式描述为 `unit_step_placeholder`。因此，当前文档不能将其写成完整线搜索算法已形成稳定公开能力。

###### 最小示例

```python
step = StepDef(
    name="step-nl",
    procedure_type="static_nonlinear",
    boundary_names=("bc-left", "bc-right"),
    output_request_names=("field-u", "field-s", "history-time"),
    parameters={
        "max_increments": 8,
        "initial_increment": 0.125,
        "min_increment": 0.125,
        "max_iterations": 20,
        "residual_tolerance": 1.0e-10,
        "displacement_tolerance": 1.0e-10,
        "allow_cutback": True,
        "line_search": False,
        "nlgeom": True,
    },
)
```

### 7. procedures 层如何调用 solver 层

`procedures` 层不会替代 solver。它的正式主线是：

```text
StepDef
    -> ProcedureBuildRequest
    -> ProcedureProvider.build(...)
    -> Assembler + DiscreteProblem + SciPyBackend
    -> ProcedureRuntime.run(...)
```

不同过程家族对 solver 的调用重点不同：

- `StaticLinearProcedure` 主要调用 `assemble_tangent()`、`assemble_external_load()`、`apply_constraints()` 和 `solve_linear_system()`。
- `ModalProcedure` 主要调用 `reduce_matrix()` 与 `solve_generalized_eigenproblem()`。
- `ImplicitDynamicProcedure` 主要调用 `assemble_mass()`、`assemble_damping()`、`assemble_tangent()`。
- `StaticNonlinearProcedure` 则围绕切线、残差、约束和状态提交组织迭代循环。

因此，过程层负责“什么时候调用 solver、调用哪些 solver 操作”，solver 层负责“这些操作怎样形成离散系统并得到数值结果”。

### 8. 多步状态继承与 `step_state_snapshots`

`CompiledModel.step_state_snapshots` 是过程层和编译产物之间一个非常关键但容易误解的接口。当前可确认的正式主线是：

1. `CompiledModel.step_state_snapshots` 以 `(channel, step_name)` 形式保存 `RuntimeState`。
2. `resolve_inherited_state()` 通过 `CompiledModel.resolve_inherited_step_state()` 读取前序状态。
3. `publish_problem_state_to_following_steps()` 通过 `CompiledModel.publish_step_state()` 发布当前状态。
4. `StepProcedureRuntime.get_state_transfer_channel()` 基类默认返回 `None`。
5. 当前源码中能直接确认重写该通道的正式过程是 `StaticNonlinearProcedure`，其通道名为 `solid_mechanics_history`。

因此，帮助文档应采用保守口径：

- 多步状态继承能力已经存在。
- 但当前最明确、最正式的继承主线主要是非线性固体力学历史状态通道。
- 不能把它外推成所有过程家族都共享完全一致的跨步继承语义。

### 9. 对象构造、引用与编译短例

```python
step = StepDef(
    name="Step-1",
    procedure_type="static_linear",
    boundary_names=("BC-FIX",),
    nodal_load_names=("LOAD-TIP",),
    output_request_names=("field-u",),
)

compiled = Compiler().compile(model)
procedure = compiled.get_step_runtime("Step-1")
```

这段示例对应三个连续动作：

1. `StepDef` 在定义层只保存过程类型和对象引用名。
2. `Compiler.compile()` 之后，这个定义对象会通过 procedure provider 映射为具体的 `ProcedureRuntime`。
3. `compiled.get_step_runtime("Step-1")` 返回的是过程运行时对象，而不是原始定义对象。

### 10. 边界对照表

下表用于澄清本层最容易混淆的相邻对象或相邻层。

| 对象或边界对 | 所属层 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- | --- |
| `StepDef` vs `ProcedureRuntime` | 定义层 / procedures 层 | 一个负责定义，一个负责执行 | 不是同一对象的两个视图 | 都带有步骤名和过程类型 |
| procedures vs kernel | procedures / kernel | 一个负责编排步骤，一个负责局部物理 | 不是同一运行时层内部子表 | 常被统称成“runtime”而混桶 |
| procedures vs solver | procedures / solver | 一个决定步骤主线，一个负责离散求解 | 不是谁包含谁的关系 | 都会接触状态和矩阵 |
| `ProcedureProvider` | procedures 层 | 过程工厂与校验入口 | 不是 `ProcedureRuntime` 本体 | 名称里有 provider 容易误认成 registry |
| `step_state_snapshots` | 编译产物 / procedures 边界 | 跨步状态快照存储点 | 不是结果数据库 | 都会保存步骤级信息 |

### 11. 本章主线图

```text
StepDef
    -> ProcedureProvider
    -> ProcedureRuntime
    -> DiscreteProblem / Backend
    -> ResultsWriter
    -> ResultStep / ResultFrame / ResultHistorySeries / ResultSummary
```

## 第 10 章 Kernel 层
本章把 `kernel` 作为独立体系讲清楚。它不再只是“编译后有哪些字段”的附属说明，而是 pyFEM 正式运行时结构中的独立层。

### 1. 这一层为什么存在

`kernel` 层解决的是“局部物理对象是什么”。它把定义层里的材料、截面、单元、约束和相互作用，翻译成 solver 可以直接读取的 runtime 对象。

这一层之所以不能继续混进“运行时层大桶”，原因有三点：

1. 它不负责步骤推进，这件事属于 procedures 层。
2. 它不负责全局矩阵装配和线性代数求解，这件事属于 solver 层。
3. 它只定义局部物理对象及其最小接口，并提供局部矩阵、向量和状态语义。

### 2. Kernel Runtime 家族地图

下表先给出 kernel 层对象家族地图，帮助读者先建立层内分工。

| 家族 | 解决的问题 | 核心对象 | 对相邻层提供什么 |
| --- | --- | --- | --- |
| 材料 runtime 家族 | 给定应变与历史状态，返回应力、切线和更新后的材料状态 | `MaterialRuntime`、`ElasticIsotropicRuntime`、`J2PlasticityRuntime` | 材料更新结果与材料点状态 |
| 截面 runtime 家族 | 绑定材料并提供截面类型语义 | `SectionRuntime`、`BeamSectionRuntime`、`PlaneStressSectionRuntime`、`PlaneStrainSectionRuntime`、`SolidSectionRuntime` | 截面类型、材料绑定、几何参数 |
| 单元 runtime 家族 | 计算局部切线、残量、质量、阻尼和单元输出 | `ElementRuntime`、`B21Runtime`、`CPS4Runtime`、`C3D8Runtime` | `ElementContribution`、质量矩阵、单元输出 |
| 约束 runtime 家族 | 描述受约束自由度及其目标值 | `ConstraintRuntime`、`DisplacementConstraintRuntime` | `ConstrainedDof` 集合 |
| 相互作用 runtime 家族 | 描述当前已注册的相互作用运行时对象 | `InteractionRuntime`、`NoOpInteractionRuntime` | 相互作用描述和状态入口 |

### 3. Kernel 与相邻层边界

| 边界对 | kernel 层负责什么 | kernel 层不负责什么 | 当前正式接口体现 |
| --- | --- | --- | --- |
| kernel vs 编译层 | 接收 provider 构造完成的 runtime 对象 | 不解析名称、不建立 DOF、不决定步骤顺序 | `Compiler` 构造 runtime，`CompiledModel` 持有 runtime |
| kernel vs procedures 层 | 提供局部物理计算接口 | 不决定分析步如何推进、不写结果 session | procedures 负责组织步骤执行与结果写出 |
| kernel vs solver 层 | 提供局部切线、残量、质量、阻尼、约束和表面载荷能力 | 不形成全局矩阵、不施加全局约束、不解线性系统 | `Assembler` 调用 `ElementRuntime` / `ConstraintRuntime` |
| kernel vs 产品壳层 | 作为内部正式运行时主线的一部分存在 | 不直接作为 GUI / API 的首选入口对象 | 普通用户通常通过 job、API、results 层间接访问 |

### 4. Runtime 家族总览

下表只回答“kernel 家族内部有哪些核心对象以及各自承担什么职责”。

| 对象 | 对象类型 | 家族内角色 | 主要职责 | 典型调用方 |
| --- | --- | --- | --- | --- |
| `MaterialRuntime` | runtime 抽象类 | 材料家族根接口 | 更新材料应力、切线与状态 | 单元 runtime |
| `SectionRuntime` | runtime 抽象类 | 截面家族根接口 | 提供截面类型与材料绑定语义 | 单元 runtime |
| `ElementRuntime` | runtime 抽象类 | 单元家族根接口 | 返回局部刚度、残量、质量、阻尼和输出 | `Assembler`、post 服务 |
| `ConstraintRuntime` | runtime 抽象类 | 约束家族根接口 | 返回受约束 DOF 集合 | `Assembler` |
| `InteractionRuntime` | runtime 抽象类 | 相互作用家族根接口 | 提供相互作用类型与描述 | 当前主线中影响很轻 |

### 5. 核心对象

#### 5.1 `MaterialRuntime`

- 所属层：kernel 层
- 家族：材料 runtime 家族
- 职责：根据应变、更新模式和已有状态返回材料响应。
- 核心字段：当前以抽象接口为主，无统一字段合同。
- 相邻对象关系：由 `MaterialDef` 经 provider 构造，通常由 `ElementRuntime` 读取。
- 典型误解：它不是步骤对象，也不是结果对象。

当前正式公开接口如下。

| 方法 | 返回 | 作用 | 谁会调用 |
| --- | --- | --- | --- |
| `get_name()` | `str` | 返回材料 runtime 名称 | 单元 runtime、调试工具 |
| `get_material_type()` | `str` | 返回材料类型键 | 单元 runtime、procedures 边界校验 |
| `allocate_state()` | `dict[str, Any]` | 分配材料点状态容器 | `StateManager`、单元 runtime |
| `update(strain, state, mode)` | `MaterialUpdateResult` | 更新应力、切线和状态 | 单元 runtime |
| `get_density()` | `float` | 返回密度 | 单元质量矩阵 |
| `describe()` | `Mapping[str, Any]` | 返回可序列化描述 | 调试、诊断、展示 |

当前可直接确认的正式实现类如下。

| 实现类 | 主要角色 | 当前正式边界 |
| --- | --- | --- |
| `ElasticIsotropicRuntime` | 各向同性线弹性材料 runtime | 覆盖 `solid`、`plane_stress`、`plane_strain`、`uniaxial` 及对应 TL 子集 |
| `J2PlasticityRuntime` | 各向同性硬化 J2 材料 runtime | 正式支持 `solid`、`plane_strain`、`uniaxial/beam_axial`；显式拒绝 `plane_stress` |

#### 5.2 `SectionRuntime`

- 所属层：kernel 层
- 家族：截面 runtime 家族
- 职责：表达截面类型，并把材料 runtime 与截面级几何参数绑定到一起。
- 核心字段：当前以抽象接口为主，无统一字段合同。
- 相邻对象关系：由 `SectionDef` 经 provider 构造，通常由 `ElementRuntime` 读取。
- 典型误解：它不是定义层 `SectionDef` 的同义词，而是编译后的截面对象。

当前正式公开接口如下。

| 方法 | 返回 | 作用 | 谁会调用 |
| --- | --- | --- | --- |
| `get_name()` | `str` | 返回截面 runtime 名称 | 单元 runtime、调试工具 |
| `get_section_type()` | `str` | 返回截面类型键 | 单元 runtime、边界校验 |
| `describe()` | `Mapping[str, Any]` | 返回可序列化描述 | 调试、诊断、展示 |

需要特别注意：`SectionRuntime` 基类本身并不强制规定 `get_material_runtime()`、`get_thickness()`、`get_area()` 这类方法，但当前正式实现类会按各自类型提供相应访问器。

当前可直接确认的正式实现类如下。

| 实现类 | 主要角色 | 当前正式边界 |
| --- | --- | --- |
| `BeamSectionRuntime` | 二维梁截面 runtime | 绑定材料、`area`、`moment_inertia_z` |
| `PlaneStressSectionRuntime` | 平面应力截面 runtime | 绑定材料与 `thickness` |
| `PlaneStrainSectionRuntime` | 平面应变截面 runtime | 绑定材料与 `thickness` |
| `SolidSectionRuntime` | 三维实体截面 runtime | 绑定材料，几何参数主要留在 `parameters` |

#### 5.3 `ElementRuntime`

- 所属层：kernel 层
- 家族：单元 runtime 家族
- 职责：返回单元局部切线、残量、质量、阻尼、表面载荷和单元输出。
- 核心字段：当前以抽象接口为主，无统一字段合同。
- 相邻对象关系：由 `ElementRecord`、`SectionRuntime`、`MaterialRuntime` 共同构造，并由 `Assembler` 和结果构造逻辑调用。
- 典型误解：它不装配全局系统，它只提供局部贡献。

当前正式公开接口如下。

| 方法 | 返回 | 作用 | 谁会调用 |
| --- | --- | --- | --- |
| `get_type_key()` | `str` | 返回单元类型键 | `Assembler`、procedures 边界校验 |
| `get_dof_layout()` | `tuple[str, ...]` | 返回每节点自由度布局 | 编译与调试工具 |
| `get_location()` | `ElementLocation` | 返回 canonical 单元位置 | `Assembler`、诊断工具 |
| `get_dof_indices()` | `tuple[int, ...]` | 返回全局 DOF 编号 | `Assembler` |
| `allocate_state()` | `dict[str, Any]` | 分配单元级状态容器 | `StateManager` |
| `compute_tangent_and_residual()` | `ElementContribution` | 计算局部切线与残量 | `Assembler` |
| `compute_mass()` | `Matrix` | 计算局部质量矩阵 | `Assembler` |
| `compute_damping()` | `Matrix \| None` | 计算局部阻尼矩阵 | `Assembler` |
| `collect_output()` | `Mapping[str, Any]` | 收集单元输出字段 | `Assembler.collect_element_outputs()`、post 服务 |
| `get_supported_geometric_nonlinearity_modes()` | `tuple[str, ...]` | 返回单元支持的几何非线性模式 | procedures 边界校验 |
| `compute_surface_load()` | `Vector` | 计算局部表面分布载荷等效节点力 | `Assembler` |

当前可直接确认的正式实现类如下。

| 实现类 | 每节点自由度 | 当前正式边界 |
| --- | --- | --- |
| `B21Runtime` | `UX/UY/RZ` | 梁单元，正式几何非线性模式为 `corotational` |
| `CPS4Runtime` | `UX/UY` | 平面连续体单元，正式 TL 主线受材料组合限制 |
| `C3D8Runtime` | `UX/UY/UZ` | 三维实体单元，当前唯一形成正式表面压力载荷主线 |

#### 5.4 `ConstraintRuntime`

- 所属层：kernel 层
- 家族：约束 runtime 家族
- 职责：描述当前约束涉及的自由度集合及其目标值。
- 核心字段：当前以抽象接口为主，无统一字段合同。
- 相邻对象关系：由 `BoundaryDef` 经 provider 构造，并由 `Assembler` 收集后施加到全局系统。
- 典型误解：它不是求解器本体，只是约束数据的 runtime 表达。

当前正式公开接口如下。

| 方法 | 返回 | 作用 | 谁会调用 |
| --- | --- | --- | --- |
| `get_name()` | `str` | 返回约束 runtime 名称 | `Assembler` |
| `get_constraint_type()` | `str` | 返回约束类型键 | `Assembler`、诊断工具 |
| `collect_constrained_dofs()` | `tuple[ConstrainedDof, ...]` | 返回受约束 DOF 与目标值集合 | `Assembler` |
| `describe()` | `Mapping[str, Any]` | 返回可序列化描述 | 调试、诊断 |

当前可直接确认的正式实现类是 `DisplacementConstraintRuntime`，它对应当前正式主线里的位移边界条件。

#### 5.5 `InteractionRuntime`

- 所属层：kernel 层
- 家族：相互作用 runtime 家族
- 职责：表达当前已注册的相互作用 runtime 对象。
- 核心字段：当前以抽象接口为主，无统一字段合同。
- 相邻对象关系：由 `InteractionDef` 经 provider 构造，并通过状态分配与描述接口进入正式执行范围。
- 典型误解：当前相互作用家族在正式主线中的作用较轻，不应把它扩写成已形成完整接触框架。

当前正式公开接口如下。

| 方法 | 返回 | 作用 | 谁会调用 |
| --- | --- | --- | --- |
| `get_name()` | `str` | 返回相互作用 runtime 名称 | 调试、诊断 |
| `get_interaction_type()` | `str` | 返回相互作用类型键 | 编译、诊断 |
| `describe()` | `Mapping[str, Any]` | 返回可序列化描述 | 调试、诊断 |

当前可直接确认的正式实现类是 `NoOpInteractionRuntime`，它对应当前最小正式 no-op 相互作用主线。

#### 5.6 `ElasticIsotropicRuntime`

`ElasticIsotropicRuntime` 是当前正式线弹性材料运行时类。该类由材料 provider 根据 `MaterialDef.material_type` 创建，供梁、平面连续体和三维实体单元在积分点或等效材料点评估应力与材料切线。

##### 数据字段

`name`  
类型为 `str`。  
该字段保存材料运行时名称。当前示例可写为 `"steel"`。

`young_modulus`  
类型为 `float`。  
该字段保存杨氏模量，例如 `2.1e11`。

`poisson_ratio`  
类型为 `float`。  
该字段保存泊松比，例如 `0.3`。

`density`  
类型为 `float`。  
默认值为 `0.0`。单元质量矩阵计算会读取该值。

##### 正式公开方法

`get_name()`  
返回 `str`。  
返回材料名称。

`get_material_type()`  
返回 `"linear_elastic"`。  
该方法返回当前材料类型键。

`allocate_state()`  
返回 `dict[str, Any]`。  
这是材料状态入口。当前初始状态字典至少包含 `strain`、`stress`、`mode`、`strain_measure`、`stress_measure`、`tangent_measure`。

`update(strain, state=None, mode="3d")`  
返回 `MaterialUpdateResult`。  
这是材料应力和材料切线的正式入口。输入 `strain` 是当前材料点评估所用的应变向量或应变表示，`state` 是上一步或上一次迭代的材料状态，`mode` 用于声明当前材料更新采用的力学模式。返回对象当前可直接确认至少包含应力、切线矩阵和更新后的状态字典。

##### 当前可直接确认的行为

当前源码可直接确认的模式集合包括 `3d`、`solid`、`plane_stress`、`plane_strain`、`uniaxial`、`beam_axial`，以及 `3d_total_lagrangian`、`solid_total_lagrangian`、`plane_stress_total_lagrangian`、`plane_strain_total_lagrangian`。

`allocate_state()` 返回的是完整可写状态字典，而不是只读描述对象。单元运行时会把该状态字典保存到材料状态映射中。

##### 当前边界

当前文档未将 `MaterialUpdateResult` 的完整字段结构展开为稳定公开合同；除应力、切线矩阵和更新状态外，其余细节应以源码实现为准。

`mode` 不是任意字符串输入。当前正式写法应以单元和截面实现实际传入的模式为准。

##### 最小示例

```python
material = ElasticIsotropicRuntime(
    name="steel",
    young_modulus=2.1e11,
    poisson_ratio=0.3,
    density=7850.0,
)
result = material.update(
    strain=(1.0e-4, 0.0, 0.0, 0.0, 0.0, 0.0),
    mode="solid",
)
```

#### 5.7 `J2PlasticityRuntime`

`J2PlasticityRuntime` 是当前正式 J2 塑性材料运行时类。该类在维持弹塑性应力更新的同时保存塑性历史变量，因此比 `ElasticIsotropicRuntime` 具有更明确的状态管理语义。

##### 数据字段

`name`  
类型为 `str`。  
该字段保存材料名称。

`young_modulus`  
类型为 `float`。  
保存杨氏模量。

`poisson_ratio`  
类型为 `float`。  
保存泊松比。

`yield_stress`  
类型为 `float`。  
保存屈服应力，例如 `250.0e6`。

`hardening_modulus`  
类型为 `float`。  
保存线性硬化模量，例如 `1.0e9`。

`density`  
类型为 `float`。  
默认值为 `0.0`。

`tangent_mode`  
类型为 `str`。  
当前可直接确认的合法值为 `"consistent"` 或 `"numerical"`。

##### 正式公开方法

`__post_init__()`  
返回 `None`。  
该方法在构造完成后执行参数校验。当前可直接确认会检查 `young_modulus > 0`、`poisson_ratio` 位于 `(-1, 0.5)`、`yield_stress > 0`、`hardening_modulus >= 0` 且 `tangent_mode` 属于允许集合。

`get_name()`  
返回 `str`。  
返回材料名称。

`get_material_type()`  
返回 `"j2_plasticity"`。  
返回当前材料类型键。

`allocate_state()`  
返回 `dict[str, Any]`。  
这是塑性材料状态入口。当前初始状态字典至少包含 `strain`、`stress`、`plastic_strain`、`equivalent_plastic_strain`、`plastic_multiplier`、`yield_function`、`is_plastic`、`mode`、`kinematic_regime`、`strain_measure`、`stress_measure`、`tangent_measure`。

`update(strain, state=None, mode="3d")`  
返回 `MaterialUpdateResult`。  
这是塑性应力更新和一致切线或数值切线的正式入口。当前帮助文档未将返回对象的完整键表冻结为稳定公开合同。

##### 当前可直接确认的行为

该类会显式维护塑性应变、等效塑性应变和屈服函数等历史量。

当前源码可直接确认的正式支持模式包括 `solid`、`plane_strain`、`uniaxial`、`beam_axial` 以及对应的 Total Lagrangian 子集。

##### 当前边界

`plane_stress` 和 `plane_stress_total_lagrangian` 当前不应写成该类的正式支持模式。

当前文档未进一步展开塑性状态字典内部所有键的演化规则；更细的增量更新细节应以源码实现为准。

##### 最小示例

```python
material = J2PlasticityRuntime(
    name="j2-steel",
    young_modulus=2.1e11,
    poisson_ratio=0.3,
    yield_stress=250.0e6,
    hardening_modulus=1.0e9,
    tangent_mode="consistent",
)
```

#### 5.8 `BeamSectionRuntime`

`BeamSectionRuntime` 是当前正式二维梁截面运行时类。`B21Runtime` 在计算轴向刚度、弯曲刚度和质量矩阵时直接读取该对象。

##### 数据字段

`name`  
类型为 `str`。  
保存截面名称，例如 `"beam-sec"`。

`material_runtime`  
类型为 `MaterialRuntime`。  
当前正式主线中通常绑定 `ElasticIsotropicRuntime` 或 `J2PlasticityRuntime`。

`area`  
类型为 `float`。  
保存截面面积，例如 `0.03`。

`moment_inertia_z`  
类型为 `float`。  
保存绕局部 `z` 轴的截面惯性矩，例如 `2.0e-4`。

`shear_factor`  
类型为 `float`。  
默认值为 `1.0`。当前帮助文档未进一步展开该参数在全部梁公式中的使用范围。

`parameters`  
类型为 `dict[str, Any]`。  
保存附加截面参数。当前文档未将完整参数键表冻结为稳定公开合同。

##### 正式公开方法

`get_name()`  
返回 `str`。  
返回截面名称。

`get_section_type()`  
返回 `"beam"`。  
返回当前截面类型键。

`get_material_runtime()`  
返回 `MaterialRuntime`。  
返回绑定的材料运行时对象。

`get_area()`  
返回 `float`。  
返回截面面积。

`get_moment_inertia_z()`  
返回 `float`。  
返回截面惯性矩。

`describe()`  
返回 `Mapping[str, Any]`。  
返回可序列化截面摘要，其中当前可直接确认至少包含 `name`、`section_type`、`material_name` 和 `parameters`。

##### 当前可直接确认的行为

`B21Runtime` 读取该对象的 `area` 和 `moment_inertia_z` 参与单元刚度和质量计算。

##### 当前边界

当前文档未进一步展开 `parameters` 中所有可选键。

##### 最小示例

```python
section = BeamSectionRuntime(
    name="beam-sec",
    material_runtime=material,
    area=0.03,
    moment_inertia_z=2.0e-4,
)
```

#### 5.9 `PlaneStressSectionRuntime` 与 `PlaneStrainSectionRuntime`

这两个类都是当前正式平面截面运行时实现。它们都向 `CPS4Runtime` 提供厚度和材料对象，但在材料更新模式上分别对应平面应力和平面应变。

##### `PlaneStressSectionRuntime`

`PlaneStressSectionRuntime` 是当前正式平面应力截面运行时类。

###### 数据字段

`name`  
类型为 `str`。  
保存截面名称。

`material_runtime`  
类型为 `MaterialRuntime`。  
绑定的材料运行时对象。

`thickness`  
类型为 `float`。  
默认值为 `1.0`。当前 `CPS4Runtime.compute_mass()` 会读取该值参与总质量计算。

`parameters`  
类型为 `dict[str, Any]`。  
保存附加截面参数。当前文档未进一步展开其稳定键表。

###### 正式公开方法

`get_name()`  
返回 `str`。  
返回截面名称。

`get_material_runtime()`  
返回 `MaterialRuntime`。  
返回绑定的材料对象。

`get_thickness()`  
返回 `float`。  
返回厚度值。

`get_section_type()`  
返回 `"plane_stress"`。  
返回平面应力截面类型键。

`describe()`  
返回 `Mapping[str, Any]`。  
返回截面摘要。

###### 当前可直接确认的行为

`CPS4Runtime` 会根据 `get_section_type()` 的返回值选择平面应力材料更新语义。

###### 当前边界

`J2PlasticityRuntime` 当前不应写成与 `PlaneStressSectionRuntime` 组合后形成正式支持主线。

###### 最小示例

```python
plane_stress = PlaneStressSectionRuntime(
    name="ps-sec",
    material_runtime=material,
    thickness=1.0,
)
```

##### `PlaneStrainSectionRuntime`

`PlaneStrainSectionRuntime` 是当前正式平面应变截面运行时类。

###### 数据字段

`name`  
类型为 `str`。  
保存截面名称。

`material_runtime`  
类型为 `MaterialRuntime`。  
绑定的材料运行时对象。

`thickness`  
类型为 `float`。  
默认值为 `1.0`。

`parameters`  
类型为 `dict[str, Any]`。  
保存附加截面参数。

###### 正式公开方法

`get_name()`  
返回 `str`。  
返回截面名称。

`get_material_runtime()`  
返回 `MaterialRuntime`。  
返回绑定的材料对象。

`get_thickness()`  
返回 `float`。  
返回厚度值。

`get_section_type()`  
返回 `"plane_strain"`。  
返回平面应变截面类型键。

`describe()`  
返回 `Mapping[str, Any]`。  
返回截面摘要。

###### 当前可直接确认的行为

当前正式 `CPS4 + J2PlasticityRuntime` 支持范围收口到平面应变路线。

###### 当前边界

当前文档未进一步展开平面应变截面的全部附加参数合同。

###### 最小示例

```python
plane_strain = PlaneStrainSectionRuntime(
    name="pe-sec",
    material_runtime=material,
    thickness=1.0,
)
```

#### 5.10 `SolidSectionRuntime`

`SolidSectionRuntime` 是当前正式三维实体截面运行时类。`C3D8Runtime` 初始化时必须绑定该对象。

##### 数据字段

`name`  
类型为 `str`。  
保存截面名称。

`material_runtime`  
类型为 `MaterialRuntime`。  
绑定的材料运行时对象。

`parameters`  
类型为 `dict[str, Any]`。  
保存附加截面参数。当前文档未进一步展开其稳定键表。

##### 正式公开方法

`get_name()`  
返回 `str`。  
返回截面名称。

`get_section_type()`  
返回 `"solid"`。  
返回实体截面类型键。

`get_material_runtime()`  
返回 `MaterialRuntime`。  
返回绑定的材料对象。

`describe()`  
返回 `Mapping[str, Any]`。  
返回可序列化摘要，当前可直接确认至少包含 `name`、`section_type`、`material_name` 和 `parameters`。

##### 当前可直接确认的行为

当前三维实体正式主线中，`C3D8Runtime` 会通过该对象读取材料运行时。

##### 当前边界

该类不是“无截面”的占位对象，不应写成可省略。

##### 最小示例

```python
solid_section = SolidSectionRuntime(
    name="solid-sec",
    material_runtime=material,
)
```

#### 5.11 `B21Runtime`

`B21Runtime` 是当前正式二维两节点梁单元运行时类。单元类型键为 `"B21"`，每个节点的自由度布局是 `("UX", "UY", "RZ")`，当前正式几何非线性模式是 `("corotational",)`。

##### 数据字段

`location`  
类型为 `ElementLocation`。  
该字段记录单元所属作用域和单元名，例如 `ElementLocation(scope_name="part-1", element_name="beam-1")`。

`coordinates`  
类型为两个二维坐标点组成的元组。  
每个坐标点写成 `(x, y)`。两组坐标按节点顺序排列。

`node_names`  
类型为长度为 `2` 的字符串元组。  
示例为 `("n1", "n2")`，顺序与 `coordinates` 一致。

`dof_indices`  
类型为整数元组。  
由于每个节点有 `UX`、`UY`、`RZ` 三个自由度，两个节点总共有 `6` 个全局自由度编号。

`section_runtime`  
类型为 `BeamSectionRuntime`。  
初始化时必须绑定梁截面对象。

`material_runtime`  
类型为 `MaterialRuntime`。  
初始化时必须绑定材料运行时对象。

##### 正式公开方法

`get_type_key()`  
返回 `"B21"`。  
返回当前单元类型键。

`get_dof_layout()`  
返回 `("UX", "UY", "RZ")`。  
返回每个节点的自由度布局。

`get_location()`  
返回 `ElementLocation`。  
返回 `location` 字段对应的单元位置对象。

`get_dof_indices()`  
返回 `tuple[int, ...]`。  
返回当前单元使用的全局自由度编号。

`get_supported_geometric_nonlinearity_modes()`  
返回 `("corotational",)`。  
该方法声明当前正式几何非线性模式集合。

`allocate_state()`  
返回 `dict[str, Any]`。  
当前初始返回值为空字典 `{}`。

`compute_tangent_and_residual(displacement=None, state=None)`  
返回 `ElementContribution`。  
这是梁单元刚度矩阵和残量向量的正式入口。

`compute_mass(state=None)`  
返回单元质量矩阵。  
当前实现按一致质量矩阵构造，比例系数为 `density * area * length / 420`。

`compute_damping(state=None)`  
返回 `None`。  
这是阻尼矩阵入口。当前该类不返回显式阻尼矩阵。

`collect_output(...)`  
返回 `Mapping[str, Any]`。  
这是梁单元输出入口。当前源码可直接确认结果至少包括 `reference_length`、`current_length`、`rigid_rotation`、`local_rotation_i`、`local_rotation_j`、`equivalent_plastic_strain`，并包含梁的轴向量和端弯矩量。

##### 当前可直接确认的行为

该类支持 `corotational` 几何非线性路线。

输出字段当前可直接确认至少包括 `axial_strain`、`axial_stress`、`axial_force`、`end_moment_i`、`end_moment_j`。

##### 当前边界

当前帮助文档未将 `collect_output()` 返回映射的完整键集合冻结为稳定公开合同。

当前源码中未直接确认 `B21Runtime` 具有正式 `compute_surface_load()` 入口，因此不能把表面压力主线写到该类名下。

##### 最小示例

```python
element = B21Runtime(
    location=ElementLocation(scope_name="part-1", element_name="beam-1"),
    coordinates=((0.0, 0.0), (2.0, 0.0)),
    node_names=("n1", "n2"),
    dof_indices=(0, 1, 2, 3, 4, 5),
    section_runtime=section,
    material_runtime=material,
)
```

#### 5.12 `CPS4Runtime`

`CPS4Runtime` 是当前正式四节点平面连续体单元运行时类。单元类型键为 `"CPS4"`，每个节点的自由度布局是 `("UX", "UY")`，当前正式几何非线性模式是 `("total_lagrangian",)`。

##### 数据字段

`location`  
类型为 `ElementLocation`。  
记录单元所属作用域和单元名。

`coordinates`  
类型为 `4` 个二维坐标点组成的元组。  
每个坐标点写成 `(x, y)`。四组坐标按节点顺序排列。

`node_names`  
类型为长度为 `4` 的字符串元组。  
顺序与 `coordinates` 一致。

`dof_indices`  
类型为整数元组。  
每个节点有 `UX` 和 `UY` 两个自由度，因此一个 `CPS4` 单元通常对应 `8` 个全局自由度编号。

`section_runtime`  
类型为 `PlaneStressSectionRuntime | PlaneStrainSectionRuntime`。  
初始化时必须绑定平面应力或平面应变截面对象。

`material_runtime`  
类型为 `MaterialRuntime`。  
初始化时必须绑定材料运行时对象。

##### 正式公开方法

`get_type_key()`  
返回 `"CPS4"`。  
返回单元类型键。

`get_dof_layout()`  
返回 `("UX", "UY")`。  
返回每个节点的自由度布局。

`get_location()`  
返回 `ElementLocation`。  
返回 `location` 字段对应的位置对象。

`get_dof_indices()`  
返回 `tuple[int, ...]`。  
返回全局自由度编号。

`get_supported_geometric_nonlinearity_modes()`  
返回 `("total_lagrangian",)`。  
声明当前正式几何非线性模式。

`allocate_state()`  
返回 `dict[str, Any]`。  
当前初始返回值为空字典 `{}`。

`compute_tangent_and_residual(displacement=None, state=None)`  
返回 `ElementContribution`。  
这是单元切线矩阵和残量向量的正式入口。当前实现会根据 `nlgeom` 标志在小变形和 Total Lagrangian 路线之间切换。

`compute_mass(state=None)`  
返回单元质量矩阵。  
当前实现使用 `density * thickness * reference_area` 计算总质量，再按四个节点平均分配，形成对角 `8 x 8` 结点集中的质量矩阵。

`compute_damping(state=None)`  
返回 `None`。  
这是阻尼矩阵接口。当前该类未形成显式阻尼矩阵公开合同。

`collect_output(...)`  
返回 `Mapping[str, Any]`。  
这是单元输出入口。当前源码可直接确认最小稳定输出键至少包括 `strain`、`stress`、`thickness`、`integration_points`、`nlgeom_active`、`nlgeom_mode`、`strain_measure`、`stress_measure`、`tangent_measure` 以及 `recovery` 元数据。

##### 当前可直接确认的行为

当前输出中包含 `averaging_weight` 和恢复所需的 `recovery` 元数据。`recovery` 中当前源码可直接确认至少包含 `target_keys`、`base_target_keys`、`extrapolation_matrix`。

`section_runtime.get_section_type()` 会影响材料更新所采用的平面应力或平面应变模式。

##### 当前边界

当前正式几何非线性子集收口到 `total_lagrangian`，不应扩写成任意几何非线性算法。

`PlaneStressSectionRuntime + J2PlasticityRuntime` 当前不应写成正式支持组合。

当前文档未将 `compute_surface_load()` 写成该类的稳定公开接口，不能把平面表面压力主线写成 `CPS4Runtime` 的正式能力。

##### 最小示例

```python
element = CPS4Runtime(
    location=ElementLocation(scope_name="part-1", element_name="plate-1"),
    coordinates=((0.0, 0.0), (1.0, 0.0), (1.0, 1.0), (0.0, 1.0)),
    node_names=("n1", "n2", "n3", "n4"),
    dof_indices=tuple(range(8)),
    section_runtime=plane_strain,
    material_runtime=material,
)
```

#### 5.13 `C3D8Runtime`

`C3D8Runtime` 是当前正式八节点三维实体单元运行时类。它属于 `ElementRuntime` 家族，单元类型键为 `"C3D8"`，每个节点的自由度布局是 `("UX", "UY", "UZ")`。当前正式几何非线性模式是 `("total_lagrangian",)`。当前帮助文档已可直接确认，它是当前唯一形成正式表面压力载荷主线的单元实现。

##### 数据字段

`location`  
类型是 `ElementLocation`。  
它记录该单元所属的作用域和单元名。当前示例可写成 `ElementLocation(scope_name="part-1", element_name="block-1")`。

`coordinates`  
类型是 `8` 个三维坐标点组成的元组。  
每个坐标点写成 `(x, y, z)`。这 `8` 组坐标按节点顺序排列，对应一个 `C3D8` 单元的 `8` 个节点。

`node_names`  
类型是长度为 `8` 的字符串元组。  
它保存这 `8` 个节点的名称，顺序应与 `coordinates` 一致。

`dof_indices`  
类型是整数元组。  
由于每个节点有 `UX`、`UY`、`UZ` 三个自由度，一个 `C3D8` 单元通常对应 `24` 个全局自由度编号。

`section_runtime`  
类型是 `SolidSectionRuntime`。  
这表示 `C3D8Runtime` 初始化时必须绑定实体截面对象。

`material_runtime`  
类型是 `MaterialRuntime`。  
这表示 `C3D8Runtime` 初始化时必须绑定材料运行时对象。

##### 正式公开方法

`get_type_key()`  
返回 `"C3D8"`。  
该方法返回当前运行时对象的单元类型键。

`get_dof_layout()`  
返回 `("UX", "UY", "UZ")`。  
该方法返回每个节点的自由度布局。

`get_location()`  
返回 `ElementLocation`。  
返回 `location` 字段对应的位置对象。

`get_dof_indices()`  
返回 `tuple[int, ...]`。  
返回 `24` 个全局自由度编号。

`get_supported_geometric_nonlinearity_modes()`  
返回 `("total_lagrangian",)`。  
这表示当前正式几何非线性路线收口在 Total Lagrangian。

`allocate_state()`  
返回 `dict[str, Any]`。  
当前这一版实现中，初始返回值为空字典 `{}`。

`compute_tangent_and_residual(displacement=None, state=None)`  
返回 `ElementContribution`。  
这是单元切线矩阵和单元残量向量的正式入口。当前实现中，该方法会先整理位移输入，再根据是否开启 `nlgeom`，在小变形路线与 Total Lagrangian 路线之间切换。

`compute_mass(state=None)`  
返回单元质量矩阵。  
当前实现中，质量矩阵按总质量平均分到 `8` 个节点的方式构造。总质量等于材料密度乘参考体积，结果是一个 `24 x 24` 的结点集中的质量矩阵。

`compute_damping(state=None)`  
返回 `None`。  
这是阻尼矩阵接口。当前帮助文档未将 `C3D8Runtime` 的具体阻尼实现展开为稳定公开合同。

`collect_output(...)`  
返回 `Mapping[str, Any]`。  
这是单元输出入口。当前源码可直接确认结果映射至少包含 `strain`、`stress`、`integration_points`、`nlgeom_active`、`nlgeom_mode`、`strain_measure`、`stress_measure`、`tangent_measure`，以及 `recovery` 元数据。当前字段中还可直接确认 `type_key`、`location`、`scope_name`、`node_names`、`node_keys`、`section_name`、`section_type`、`material_name`、`averaging_weight`。

`compute_surface_load(local_face, load_type, components, state=None)`  
返回等效节点力向量。  
这是表面分布载荷入口。当前正式支持范围收口到小变形 `pressure` 或 `p` 表面载荷。

##### 当前可直接确认的行为

`compute_tangent_and_residual()` 会在小变形路线与 Total Lagrangian 路线之间切换。

`compute_mass()` 按参考体积和密度构造结点集中的质量矩阵。

`collect_output()` 当前已直接确认至少可返回 `strain` 和 `stress`，并保留积分点数据与恢复元数据。

`compute_surface_load()` 已形成正式主线，但当前只适用于小变形 `pressure` 表面载荷。

##### 当前边界

当前仅正式支持小变形 `pressure` 表面载荷。

`nlgeom=True` 下的表面分布载荷不应写成当前已支持能力。

`follower pressure` 当前不应写成已支持能力。

当前帮助文档未将完整输出键表展开为稳定公开合同。

##### 最小示例

```python
element = C3D8Runtime(
    location=ElementLocation(scope_name="part-1", element_name="block-1"),
    coordinates=(
        (0.0, 0.0, 0.0),
        (1.0, 0.0, 0.0),
        (1.0, 1.0, 0.0),
        (0.0, 1.0, 0.0),
        (0.0, 0.0, 1.0),
        (1.0, 0.0, 1.0),
        (1.0, 1.0, 1.0),
        (0.0, 1.0, 1.0),
    ),
    node_names=("n1", "n2", "n3", "n4", "n5", "n6", "n7", "n8"),
    dof_indices=tuple(range(24)),
    section_runtime=solid_section,
    material_runtime=material,
)
```

#### 5.14 `DisplacementConstraintRuntime`

`DisplacementConstraintRuntime` 是当前唯一正式约束 runtime。它的角色不是“再解释一遍边界条件”，而是把定义层边界条件变成 solver 能直接收集的受约束自由度集合。

- 当前可确认的核心字段：
  `name`、`boundary_type`、`target_name`、`target_type`、`scope_name`、`constrained_dofs`、`dof_values`、`parameters`
- `collect_constrained_dofs()` 返回：
  `tuple[ConstrainedDof, ...]`
- `describe()` 会保留约束目标、作用域和规定值等摘要

帮助文档在这里最该提醒读者的是：

- `BoundaryDef.dof_values` 还是定义层输入；
- `DisplacementConstraintRuntime.constrained_dofs` 才是 solver 实际施加约束时读取的 DOF 集合。

#### 5.15 `NoOpInteractionRuntime`

`NoOpInteractionRuntime` 是当前最小正式相互作用实现。它存在的意义是把 interaction 家族纳入正式运行时结构，但不把未来接触、耦合、tie 等能力提前写成已支持。

- 当前可确认的核心字段：
  `name`、`interaction_type`、`scope_name`、`parameters`
- 当前公开方法：
  `get_name()`、`get_interaction_type()`、`describe()`

这类对象在帮助文档里的正确写法是：

- 它已经是正式对象家族中的一个最小实现；
- 但它不代表接触主线已经完整形成。

### 6. 当前正式实现家族

下表把当前 kernel 层已进入正式主线的实现家族收束到一张总览表中。

| 家族 | 当前正式实现 | 当前正式边界 |
| --- | --- | --- |
| 材料 | `ElasticIsotropicRuntime`、`J2PlasticityRuntime` | 不扩大写成超弹、黏弹、损伤等其他材料族 |
| 截面 | `BeamSectionRuntime`、`PlaneStressSectionRuntime`、`PlaneStrainSectionRuntime`、`SolidSectionRuntime` | 不扩大写成壳截面或任意高阶截面体系 |
| 单元 | `B21Runtime`、`CPS4Runtime`、`C3D8Runtime` | 不扩大写成壳单元、三角形、四面体或高阶实体族 |
| 约束 | `DisplacementConstraintRuntime` | 当前正式边界是位移型边界条件 |
| 相互作用 | `NoOpInteractionRuntime` | 当前不把它写成完整接触或 tie 主线 |

### 7. provider 与 placeholder runtime 在 kernel 层中的含义

这里需要明确区分两层概念：

1. provider 绑定发生在编译层。
2. 被绑定出来的正式实现或 placeholder，形状上属于 kernel runtime 家族。

当前编译主线会通过 `RuntimeRegistry` 为材料、截面、单元、约束和相互作用分别查找 provider。若命中 provider，就生成正式 kernel runtime；若未命中，就生成与该家族对应的 placeholder runtime，例如：

- `MaterialRuntimePlaceholder`
- `SectionRuntimePlaceholder`
- `ElementRuntimePlaceholder`
- `ConstraintRuntimePlaceholder`
- `InteractionRuntimePlaceholder`

对 placeholder runtime 的正式口径保持保守：

1. 它表示“编译结构保留了这个位置”。
2. 它不等于“该家族能力已正式支持”。
3. 文档不得把 placeholder runtime 当成稳定用户接口。

### 8. 对象使用短例

下面的例子只演示 kernel 对象如何进入 `CompiledModel` 并在后续执行中被读取。

```python
compiled = Compiler().compile(model)

material_runtime = compiled.material_runtimes["Steel"]
section_runtime = compiled.section_runtimes["BEAM_SEC"]
element_runtime = compiled.get_element_runtime("BeamPart.E1")
constraint_runtime = compiled.constraint_runtimes["BC-FIX"]
```

这条链说明：

1. `MaterialDef` 不会被 solver 直接读取，solver 读取的是 `MaterialRuntime` 参与形成的单元贡献。
2. `SectionDef` 在编译后变成 `SectionRuntime`，再由单元 runtime 读取。
3. `ElementRecord` 在编译后变成 `ElementRuntime`，这是 solver 直接装配的局部对象。

### 9. 边界对照表

| 对象或边界对 | 所属层 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- | --- |
| `MaterialDef` vs `MaterialRuntime` | 定义层 / kernel 层 | 前者是定义，后者是材料运行时 | 同一对象的两种别名 | 编译前后边界经常被写混 |
| `SectionDef` vs `SectionRuntime` | 定义层 / kernel 层 | 前者定义“截面是什么”，后者提供可执行截面语义 | 同一层对象 | runtime 才会被单元直接读取 |
| `ElementRuntime` vs `Assembler` | kernel / solver | 前者计算局部贡献，后者装配全局系统 | 都不是对方的别名 | 局部与全局职责不同 |
| `ConstraintRuntime` vs `BoundaryDef` | kernel / 定义层 | 前者是编译后受约束 DOF 集合，后者是定义对象 | 同一对象 | `target_name` 解析发生在编译层，不在 runtime 内重新做 |
| placeholder runtime | 编译后占位对象 | provider 缺失时的受控占位结构 | 正式功能完备实现 | 不能据此宣称已支持该能力 |

### 10. 本章主线图

```text
SectionDef / MaterialDef / ElementRecord / BoundaryDef / InteractionDef
    -> provider 绑定
    -> kernel runtimes
    -> Assembler / procedures / post 服务
```

---

## 第 11 章 Solver 层
本章把 `solver` 作为独立体系讲清楚。它负责形成全局离散系统、施加约束、调用线性代数后端并管理 trial / committed 状态。

### 1. 这一层为什么存在

`solver` 层解决的是“局部 runtime 贡献怎样变成全局数值问题”。它之所以不能继续隐藏在 procedures 或 kernel 里，原因有四点：

1. `kernel` 只提供局部物理对象和局部贡献，不负责全局系统装配。
2. `procedures` 负责步骤编排，不直接承担矩阵求解和状态管理实现。
3. `solver` 需要统一处理切线矩阵、质量矩阵、阻尼矩阵、约束和自由度约化。
4. `RuntimeState` 与 `StateManager` 提供的是求解内部状态，而不是结果层对象。

因此，`solver` 层应被正式写成“离散系统与状态推进层”，而不是继续被称作运行时层里的一组内部细节。

### 2. Solver 家族地图

下表先给出本层对象家族地图，帮助读者先建立层内分工，再进入具体对象。

| 家族 | 所属层 | 解决的问题 | 核心对象 | 与相邻对象关系 |
| --- | --- | --- | --- | --- |
| 装配家族 | solver 层 | 把 kernel runtime 的局部贡献装配成全局离散对象 | `Assembler`、`ReducedSystem` | 读取 kernel runtime，并把结果交给 `DiscreteProblem` 和 backend |
| 问题包装家族 | solver 层 | 对外暴露统一的离散问题接口 | `DiscreteProblem` | 由 procedures 持有，并调用 `Assembler` 和 `StateManager` |
| 线性代数后端家族 | solver 层 | 求解线性系统和广义特征值问题 | `LinearAlgebraBackend`、`SciPyBackend` | 由 `DiscreteProblem` 使用方或过程对象持有；不直接面向普通用户 |
| 状态家族 | solver 层 | 管理全局位移/速度/加速度与局部状态映射 | `GlobalKinematicState`、`RuntimeState`、`StateManager` | 依据 `CompiledModel` 分配，并由 procedures 读取后构造结果 |

### 3. solver 层与相邻层的边界

下表用于澄清最容易混淆的相邻层。

| 对象或边界对 | 所属层 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- | --- |
| solver vs kernel | solver / kernel | 一个负责全局离散系统，一个负责局部物理对象 | 不是同一运行时层的两个名称 | 两者都处理 state，容易被合并成 runtime |
| solver vs procedures | solver / procedures | 一个提供求解操作，一个决定步骤主线 | 不是过程层内部私有实现 | 过程代码里会调用 solver 方法 |
| solver vs API / GUI / job | solver / 壳层 | solver 是内部数值层 | 不是普通用户稳定依赖的入口层 | 顶层 API 最终会用到 solver，但不应暴露其内部对象 |

### 4. 核心对象

#### 4.1 `Assembler`

`Assembler` 是当前正式装配类。它读取 `CompiledModel` 中的单元、约束和载荷定义，生成全局刚度矩阵、残量向量、质量矩阵、阻尼矩阵以及受约束系统。

##### 数据字段

`_compiled_model`  
类型为 `CompiledModel`。  
构造函数接收该对象，并在后续装配过程中访问 `element_runtimes`、`constraint_runtimes`、`dof_manager` 以及定义层载荷对象。

`_nlgeom`  
类型为 `bool`。  
构造函数参数 `nlgeom` 会写入该字段。当前 `assemble_tangent()` 和 `assemble_residual()` 所调用的单元状态解析逻辑会读取这一标志。

##### 正式公开方法

`assemble_tangent(state=None)`  
返回 `numpy.ndarray`。  
这是全局切线矩阵入口。当前实现会遍历全部 `element_runtimes`，对每个单元调用 `element_runtime.tangent_residual(...)`，读取返回的局部刚度矩阵并散射到全局矩阵。

`assemble_residual(state=None)`  
返回 `numpy.ndarray`。  
这是全局内力残量向量入口。当前实现同样遍历全部单元，读取 `ElementContribution.residual` 并散射到全局向量。

`assemble_mass(state=None)`  
返回 `numpy.ndarray`。  
这是全局质量矩阵入口。当前实现对每个单元调用 `element_runtime.mass(...)` 并进行全局装配。

`assemble_damping(state=None)`  
返回 `numpy.ndarray`。  
这是全局阻尼矩阵入口。当前实现对每个单元调用 `element_runtime.damping(...)`；若单元返回 `None`，则该单元不向全局阻尼矩阵贡献条目。

`assemble_external_load(step_definition, time=0.0, state=None, load_scale=1.0)`  
返回 `numpy.ndarray`。  
这是全局外载向量入口。当前实现会先处理 `step_definition.nodal_load_names`，再处理 `step_definition.distributed_load_names`，并把荷载值写入对应自由度。

`collect_constraints(boundary_names)`  
返回 `tuple[ConstrainedDof, ...]`。  
该方法从 `constraint_runtimes` 中收集位移约束自由度和值，并在发现同一自由度收到冲突值时抛出 `SolverError`。

`build_constraint_value_map(boundary_names, scale_factor=1.0)`  
返回 `dict[int, float]`。  
该方法把约束集合转换为 `dof_index -> prescribed_value` 映射。

`apply_constraints(matrix, rhs, boundary_names, scale_factor=1.0)`  
返回 `tuple[numpy.ndarray, numpy.ndarray]`。  
这是对全局线性系统施加位移约束的正式入口。当前实现内部转调 `apply_prescribed_values(...)`。

`apply_prescribed_values(matrix, rhs, prescribed_values)`  
返回 `tuple[numpy.ndarray, numpy.ndarray]`。  
该方法对任意线性系统施加给定位移值。当前实现会先从右端中扣除原矩阵列对规定值的贡献，再把对应行列清零，并把对角项改为 `1.0`。

`reduce_matrix(matrix, boundary_names)`  
返回 `ReducedSystem`。  
该方法根据位移约束筛出自由自由度，并返回只包含自由自由度子矩阵的约化系统。

`collect_element_outputs(state=None)`  
返回 `dict[str, dict[str, Any]]`。  
这是单元输出收集入口。当前返回值是 `qualified_element_name -> output_dict` 映射。

`extract_element_displacement(element_runtime, global_displacement)`  
返回 `numpy.ndarray`。  
该方法按单元自由度编号从全局位移向量中抽取局部位移子向量。

##### 当前可直接确认的行为

`assemble_tangent()` 和 `assemble_residual()` 都会把全局位移向量按单元自由度切片，再把局部位移作为 `tuple` 传入单元运行时。

`assemble_external_load()` 当前支持节点载荷和分布载荷两类定义对象，并读取每个载荷定义里 `parameters["scale"]` 的缩放系数，缺省值为 `1.0`。

`collect_constraints()` 会对重复约束自由度执行冲突检查。

##### 当前边界

`Assembler` 不是步骤对象，不写结果文件，也不直接求解线性系统。

分布载荷是否能够真正形成全局外载，仍受具体单元 `compute_surface_load()` 或等效载荷实现的当前支持范围约束。

##### 最小示例

```python
compiled = Compiler().compile(model)
assembler = Assembler(compiled, nlgeom=False)
state = DiscreteProblem(compiled, assembler=assembler).create_zero_state()

tangent = assembler.assemble_tangent(state=state)
external_load = assembler.assemble_external_load(model.steps["step-static"], state=state)
```

#### 4.2 `DiscreteProblem`

`DiscreteProblem` 是当前正式离散问题包装类。它把 `Assembler`、`StateManager` 和与步骤无关的求解前后处理接口收拢到同一个对象中，供 `ProcedureRuntime` 直接调用。

##### 数据字段

`_compiled_model`  
类型为 `CompiledModel`。  
构造函数接收该对象，并通过只读属性 `compiled_model` 暴露。

`_assembler`  
类型为 `Assembler`。  
如果构造时未显式给出，则当前实现会自动创建 `Assembler(compiled_model)`。

`_state_manager`  
类型为 `StateManager`。  
构造函数总是基于当前 `compiled_model` 创建一个状态管理器。

##### 正式公开方法

`compiled_model`  
返回 `CompiledModel`。  
返回当前关联的编译模型。

`state_manager`  
返回 `StateManager`。  
返回运行时状态管理器。

`create_zero_state(time=0.0)`  
返回 `RuntimeState`。  
创建一份带局部状态容器的零初始状态。

`begin_trial(state=None)`  
返回 `RuntimeState`。  
以给定状态或当前提交状态为基准开启试算状态。

`get_committed_state()`  
返回 `RuntimeState`。  
返回当前已提交状态的副本。

`get_trial_state()`  
返回 `RuntimeState`。  
返回当前试算状态的副本。

`set_trial_state(state)`  
返回 `None`。  
把外部提供的状态写成当前试算状态。

`assemble_tangent(state=None)`  
返回 `numpy.ndarray`。  
转调 `Assembler.assemble_tangent()`。

`assemble_residual(state=None)`  
返回 `numpy.ndarray`。  
转调 `Assembler.assemble_residual()`。

`assemble_mass(state=None)`  
返回 `numpy.ndarray`。  
转调 `Assembler.assemble_mass()`。

`assemble_damping(state=None)`  
返回 `numpy.ndarray`。  
转调 `Assembler.assemble_damping()`。

`assemble_external_load(step_definition, time=0.0, state=None, load_scale=1.0)`  
返回 `numpy.ndarray`。  
转调 `Assembler.assemble_external_load()`。

`collect_constraints(boundary_names)`  
返回 `tuple[ConstrainedDof, ...]`。  
转调装配器收集约束。

`build_constraint_value_map(boundary_names, scale_factor=1.0)`  
返回 `dict[int, float]`。  
转调装配器构造约束值映射。

`apply_constraints(matrix, rhs, boundary_names, scale_factor=1.0)`  
返回 `tuple[numpy.ndarray, numpy.ndarray]`。  
转调装配器施加位移约束。

`apply_prescribed_values(matrix, rhs, prescribed_values)`  
返回 `tuple[numpy.ndarray, numpy.ndarray]`。  
转调装配器施加给定位移值。

`reduce_matrix(matrix, boundary_names)`  
返回 `ReducedSystem`。  
根据位移约束构造约化矩阵系统。

`expand_reduced_vector(vector, free_indices, prescribed_values=None)`  
返回 `numpy.ndarray`。  
把缩减系统中的解向量扩展回全局自由度。当前文档对该方法内部写值顺序未进一步展开，具体实现应以源码为准。

`collect_element_outputs(state=None)`  
返回 `dict[str, dict[str, Any]]`。  
收集全部单元输出字典。

`commit(state=None)`  
返回 `RuntimeState`。  
提交当前试算状态。如果传入状态参数，则先把该状态写入试算状态再提交。

`rollback()`  
返回 `RuntimeState`。  
回滚到最近一次已提交状态。

##### 当前可直接确认的行为

当前 `DiscreteProblem` 不直接持有线性代数后端对象。步骤类会把 `DiscreteProblem` 和 `LinearAlgebraBackend` 并列持有。

`commit()` 和 `rollback()` 的状态复制由 `StateManager` 完成，因此过程对象读到的是状态副本而非内部原地对象。

##### 当前边界

`DiscreteProblem` 不是结果数据库对象，也不是产品层推荐直接依赖的入口。

当前文档未将所有辅助方法的完整参数默认值和异常类型冻结为稳定公开合同；更细的行为应以源码实现为准。

##### 最小示例

```python
compiled = Compiler().compile(model)
problem = DiscreteProblem(compiled)

trial = problem.begin_trial()
tangent = problem.assemble_tangent(state=trial)
load = problem.assemble_external_load(model.steps["step-static"], state=trial)
```

#### 4.3 `LinearAlgebraBackend` 与 `SciPyBackend`

- 所属层：solver 层
- 家族：线性代数后端家族
- 职责：封装线性系统与广义特征值问题的数值求解
- 核心字段：当前文档只确认接口和当前内置实现，不承诺更多内部字段
- 主要方法状态：公开方法很少，属于“以少量求解接口为主”的服务类
- 相邻对象关系：由 `ProcedureProvider` 或 `ProcedureRuntime` 选择并持有，并返回数值解向量或模态结果
- 典型误解：不要把 backend 视为完整 solver 层本身，它只是 solver 的一个子家族

下表只列当前正式公开边界的方法。

| 方法 | 所属类 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 | 常见误解 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `solve_linear_system()` | `LinearAlgebraBackend` | 矩阵、右端向量 | 解向量 | 求解线性方程组 | 线性静力、非线性迭代、动力步骤 | 过程对象、`DiscreteProblem` 使用方 | 不是矩阵装配入口 |
| `solve_generalized_eigenproblem()` | `LinearAlgebraBackend` | 刚度矩阵、质量矩阵、模态数 | 特征值与特征向量 | 求解模态特征问题 | `ModalProcedure` 中 | 过程对象 | 不是普通线性系统求解 |

当前内置后端是 `SciPyBackend`。帮助文档应把它写成“当前正式内置实现”，不要外推成多后端生态已经稳定存在。

#### 4.4 `RuntimeState` 与 `GlobalKinematicState`

`GlobalKinematicState` 和 `RuntimeState` 是当前正式状态对象。前者只保存全局运动学向量和时间，后者在其基础上增加单元、材料、积分点、相互作用和约束的局部状态映射。

##### 数据字段

`GlobalKinematicState.displacement`  
类型为 `numpy.ndarray`。  
保存全局位移向量，长度等于总自由度数。

`GlobalKinematicState.velocity`  
类型为 `numpy.ndarray`。  
保存全局速度向量，长度与位移向量一致。

`GlobalKinematicState.acceleration`  
类型为 `numpy.ndarray`。  
保存全局加速度向量，长度与位移向量一致。

`GlobalKinematicState.time`  
类型为 `float`。  
保存当前状态对应的时间或步内时刻。

`RuntimeState.kinematics`  
类型为 `GlobalKinematicState`。  
保存全局位移、速度、加速度和时间。

`RuntimeState.element_states`  
类型为 `dict[str, dict[str, Any]]`。  
键是单元限定名，例如 `part-1.e1`；值是该单元的局部状态字典。

`RuntimeState.material_states`  
类型为 `dict[str, dict[str, Any]]`。  
键是材料运行时名称，例如 `mat-1`；值是材料状态字典。

`RuntimeState.integration_point_states`  
类型为 `dict[str, dict[str, dict[str, Any]]]`。  
当前帮助文档可直接确认这是积分点状态入口。完整键结构当前未进一步展开为稳定公开合同。

`RuntimeState.interaction_states`  
类型为 `dict[str, dict[str, Any]]`。  
键是相互作用运行时名称。

`RuntimeState.constraint_states`  
类型为 `dict[str, dict[str, Any]]`。  
键是约束运行时名称，例如 `bc-root`。

##### 正式公开方法

`displacement`  
返回 `numpy.ndarray`。  
这是 `RuntimeState` 对 `kinematics.displacement` 的属性代理；写入时会复制为新的 `numpy.ndarray`。

`velocity`  
返回 `numpy.ndarray`。  
这是对全局速度向量的属性代理。

`acceleration`  
返回 `numpy.ndarray`。  
这是对全局加速度向量的属性代理。

`time`  
返回 `float`。  
这是对当前状态时刻的属性代理。

##### 当前可直接确认的行为

源码中存在 `ProblemState = RuntimeState` 别名，因此“问题状态”在当前实现中就是 `RuntimeState`。

状态对象既保存全局向量，也保存局部状态字典，过程对象提交或回滚时会同时处理这两部分。

##### 当前边界

`RuntimeState` 不是正式结果对象，不能与 `ResultFrame` 或 `ResultField` 混写。

当前帮助文档未进一步展开 `integration_point_states` 的完整嵌套键规范。

##### 最小示例

```python
state = problem.create_zero_state()
state.displacement[:] = 0.0
state.time = 0.0
state.material_states["mat-1"]["mode"] = "solid"
```

#### 4.5 `StateManager`

`StateManager` 是当前正式状态管理类。它负责按照 `CompiledModel` 分配新状态、复制状态、创建试算状态、提交状态和回滚状态。

##### 数据字段

`_compiled_model`  
类型为 `CompiledModel`。  
该字段用于查询总自由度数和全部运行时对象集合。

`_committed_state`  
类型为 `RuntimeState`。  
保存最近一次已提交状态。

`_trial_state`  
类型为 `RuntimeState`。  
保存当前试算状态。

##### 正式公开方法

`committed_state`  
返回 `RuntimeState`。  
返回内部已提交状态对象。

`trial_state`  
返回 `RuntimeState`。  
返回内部试算状态对象。

`allocate_runtime_state(time=0.0)`  
返回 `RuntimeState`。  
这是状态分配入口。当前实现会根据 `compiled_model.dof_manager.num_dofs()` 创建零位移、零速度、零加速度向量，并为单元、材料、相互作用和约束运行时分配局部状态字典。

`copy_state(state)`  
返回 `RuntimeState`。  
深拷贝一份运行时状态，包括全局向量和全部局部状态映射。

`get_committed_state()`  
返回 `RuntimeState`。  
返回已提交状态的副本。

`get_trial_state()`  
返回 `RuntimeState`。  
返回试算状态的副本。

`begin_trial(base_state=None)`  
返回 `RuntimeState`。  
以给定状态或当前已提交状态为基准创建新的试算状态。

`set_trial_state(state)`  
返回 `None`。  
用给定状态替换当前试算状态。

`commit(state=None)`  
返回 `None`。  
提交当前试算状态。如果传入状态参数，则先用该状态替换试算状态。

`rollback()`  
返回 `RuntimeState`。  
把试算状态恢复到最近一次已提交状态，并返回恢复后的试算状态副本。

##### 当前可直接确认的行为

`allocate_runtime_state()` 会遍历 `element_runtimes`、`material_runtimes`、`interaction_runtimes` 和 `constraint_runtimes`，对存在 `allocate_state()` 方法的运行时对象调用该方法，并把得到的字典副本写入状态映射。

当前实现把 `integration_point_states` 初始化为空字典 `{}`，并预留积分点状态入口。

##### 当前边界

`StateManager` 不写结果文件，也不管理多步作业顺序。

当前文档未把 `_allocate_local_state()` 的全部异常边界冻结为稳定公开合同。

##### 最小示例

```python
manager = StateManager(compiled)
trial = manager.begin_trial()
trial.time = 0.25
manager.commit()
restored = manager.rollback()
```

#### 4.6 `ReducedSystem`

- 所属层：solver 层
- 家族：装配家族
- 职责：保存约化后的系统矩阵及自由索引
- 核心字段：`matrix`、`free_indices`
- 主要方法状态：当前以数据容器为主
- 相邻对象关系：由 `Assembler.reduce_matrix()` 或 `DiscreteProblem.reduce_matrix()` 产出，用于模态或受约束线性系统
- 典型误解：它不是完整求解器对象

#### 4.7 最小执行路径示例

如果只看方法表，读者很难真正理解 solver 层对象是怎样接起来的。下面用一条最小静力路径把对象输入、输出和调用顺序串起来。

```python
compiled = Compiler().compile(model)
assembler = Assembler(compiled, nlgeom=False)
problem = DiscreteProblem(compiled, assembler=assembler)
trial = problem.begin_trial()

tangent = problem.assemble_tangent(state=trial)
external_load = problem.assemble_external_load(model.steps["step-static"], state=trial)

# 约束施加与线性求解通常继续由过程对象组织
# reduced_system = problem.apply_constraints(...)
# solution = backend.solve_linear_system(...)

problem.commit()
```

把这段最小路径拆开看，solver 层的对象关系就比较清楚了：

| 对象 | 它读什么 | 它产出什么 | 下一跳是谁 |
| --- | --- | --- | --- |
| `CompiledModel` | 编译层产物 | runtime 对象图、自由度信息、步骤运行时 | `Assembler`、`DiscreteProblem` |
| `Assembler` | `element_runtimes`、`constraint_runtimes`、当前 `RuntimeState` | 全局矩阵、全局向量、约束集合、单元输出集合 | `DiscreteProblem` |
| `DiscreteProblem` | `Assembler`、`StateManager` | 对过程层可用的统一求解接口 | `ProcedureRuntime` |
| `LinearAlgebraBackend` | 约化后的矩阵与右端 | 解向量或特征对 | `ProcedureRuntime` |
| `StateManager` | `CompiledModel`、各 runtime 的 `allocate_state()` | committed / trial 状态 | `DiscreteProblem`、过程层 |

这里最关键的不是 API 名字，而是分工：

1. `Assembler` 负责“把局部贡献放到全局位置上”。
2. `DiscreteProblem` 负责“把装配、约束和状态管理包装成过程层能调用的统一接口”。
3. `LinearAlgebraBackend` 负责“真正数值求解”。
4. `StateManager` 负责“哪些状态还在试探，哪些状态已经提交”。

如果换成模态步或动力步，主线不会变，只是过程层调用的 solver 方法组合不同：

- 模态步会重点走 `assemble_mass()`、`reduce_matrix()` 和 `solve_generalized_eigenproblem()`。
- 隐式动力会在每个时间步内反复用到 `assemble_mass()`、`assemble_damping()`、`assemble_tangent()` 和约束处理。
- 静力非线性会围绕 `assemble_tangent()`、`assemble_residual()`、`commit()`、`rollback()` 组织增量迭代。

因此，solver 层真正提供的是“步骤无关的离散问题操作集”，而不是某一种具体分析流程。

### 5. 当前正式主线里 procedures 如何调用 solver

当前正式主线中，solver 层主要由 procedure provider 组织进入步骤执行链。典型路径是：

```text
StepDef
    -> ProcedureProvider.build(...)
    -> Assembler(compiled_model, nlgeom=...)
    -> DiscreteProblem(compiled_model, assembler=...)
    -> SciPyBackend()
    -> ProcedureRuntime.run(...)
```

这说明：

1. procedures 决定什么时候创建 solver 对象。
2. solver 负责如何装配、约束、求解和管理状态。
3. kernel runtime 提供局部贡献，但不直接决定步骤推进。

### 6. 哪些对象属于内部对象

下列对象虽然已经进入正式主线，但普通用户不应把它们当成稳定的直接依赖入口：

- `Assembler`
- `ReducedSystem`
- `RuntimeState`
- `StateManager`
- `SciPyBackend`

普通用户或脚本更适合依赖的入口仍然是：

- `JobManager`
- `PyFEMSession`
- `run_model()` / `run_input_file()`
- `ResultsFacade`

这条边界很重要，因为 solver 层的对象更多面向框架内部和开发者扩展，而不是产品壳层。

### 7. 对象使用短例

```python
compiled = Compiler().compile(model)
assembler = Assembler(compiled, nlgeom=False)
problem = DiscreteProblem(compiled, assembler=assembler)
trial = problem.begin_trial()
tangent = problem.assemble_tangent(state=trial)
load = problem.assemble_external_load(model.steps["Step-1"], state=trial)
```

这段示例表达的是 solver 层内部对象的调用顺序：

1. `CompiledModel` 提供编译后的 runtime 对象图和自由度信息。
2. `Assembler` 从这些 runtime 对象中收集局部贡献。
3. `DiscreteProblem` 把装配、状态和约束处理包装成统一接口。

帮助文档应把这类示例写成开发者视角的内部主线，而不是普通用户的推荐调用方式。

### 8. 边界对照表

下表用于澄清本层最容易混淆的相邻对象或相邻层。

| 对象或边界对 | 所属层 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- | --- |
| `ElementRuntime` vs `Assembler` | kernel / solver | 一个提供局部单元行为，一个负责全局装配 | 不是同一类 runtime 的不同别名 | 都会接触单元刚度和输出 |
| `Assembler` vs `DiscreteProblem` | solver 层 | 一个负责装配，一个负责统一问题接口与状态 | 不是简单包含关系即可替代说明 | 都暴露大量求解相关方法 |
| `DiscreteProblem` vs `ProcedureRuntime` | solver / procedures | 一个负责离散问题，一个负责步骤编排 | 不是谁都能直接代替另一方 | 两者都会操作状态 |
| `RuntimeState` vs `ResultsDatabase` | solver / 结果层 | 一个是求解内部状态，一个是持久化结果结构 | 不是内存版与磁盘版的对应物 | 都保存位移等数据 |
| `SciPyBackend` vs `PyFEMSession` | solver / 产品壳层 | 一个是线性代数后端，一个是用户入口壳层 | 不是同层 API | 上层最终会间接用到 backend |

### 9. 本章主线图

```text
kernel runtimes
    -> Assembler
    -> DiscreteProblem
    -> LinearAlgebraBackend
    -> RuntimeState / StateManager
    -> procedures 层结果写出
```

---

## 第 12 章 结果层
本章把 `results` 作为独立体系讲清楚。它不再只是“结果 JSON 长什么样”的字段说明，而是 pyFEM 正式主线中的独立层：负责结果写出、结果层级、结果读取以及 facade/query/probe 的访问接口。

### 1. 这一层为什么存在

`results` 层解决的是“步骤执行之后，结果如何被保存、读取和访问”。它之所以必须独立于 procedures 和 solver，原因有四点：

1. procedures 负责组织步骤执行，但不应自己承担结果协议定义。
2. solver 负责内部状态推进，但 `RuntimeState` 不是正式结果数据库对象。
3. 结果需要稳定的层级结构，供 API、GUI、probe 和导出统一读取。
4. `ResultsFacade / Query / Probe` 属于结果访问侧，不应回头承担求解层职责。

因此，结果层的正式主线应理解为：

```text
ProcedureRuntime
    -> ResultsWriter
    -> ResultsDatabase
    -> ResultsReader
    -> ResultsFacade / Query / Probe / exporter / GUI
```

### 2. 结果对象家族地图

下表先给出本层对象家族地图，帮助读者先建立层内分工，再进入具体对象。

| 家族 | 所属层 | 解决的问题 | 核心对象 | 与相邻对象关系 |
| --- | --- | --- | --- | --- |
| 写读接口家族 | 结果层 | 定义结果如何写入与读回 | `ResultsWriter`、`ResultsReader` | 由 procedures 调用，并产出或读取 `ResultsDatabase` |
| 数据库对象家族 | 结果层 | 定义结果协议的正式层级结构 | `ResultsSession`、`ResultsDatabase` | 接收 writer 写入，并供 facade 和 reader 读取 |
| 步骤结果家族 | 结果层 | 组织一步中的帧、场、历史量和摘要 | `ResultStep`、`ResultFrame`、`ResultField`、`ResultHistorySeries`、`ResultSummary` | 由过程输出构造，并供 query/probe 读取 |
| 访问接口家族 | 结果层访问侧 | 为脚本、GUI、导出和 probe 提供 reader-only 入口 | `ResultsFacade`、`ResultsQueryService`、`ResultsProbeService` | 读取 `ResultsReader` / `ResultsDatabase`，并供 API、GUI、VTK 导出使用 |

### 3. 结果层级表

下表专门说明结果对象在数据库中的挂载层级。

| 层级 | 对象 | 上级位置 | 下级对象 | 当前含义 |
| --- | --- | --- | --- | --- |
| 0 | `ResultsDatabase` | 根对象 | `ResultsSession`、`ResultStep[]` | 一份完整结果数据库 |
| 1 | `ResultsSession` | `ResultsDatabase.session` | 无 | 结果会话元数据 |
| 1 | `ResultStep` | `ResultsDatabase.steps[*]` | `ResultFrame[]`、`ResultHistorySeries[]`、`ResultSummary[]` | 一个正式步骤的全部结果 |
| 2 | `ResultFrame` | `ResultStep.frames[*]` | `ResultField[]` | 一步中的单帧场结果 |
| 3 | `ResultField` | `ResultFrame.fields[*]` | 无 | 单个场变量及其目标值 |
| 2 | `ResultHistorySeries` | `ResultStep.histories[*]` | 无 | 历史量序列 |
| 2 | `ResultSummary` | `ResultStep.summaries[*]` | 无 | 步骤级摘要 |

正式结果层级可以概括为：

```text
ResultsDatabase
├─ session: ResultsSession
└─ steps: ResultStep[]
   ├─ frames: ResultFrame[]
   │  └─ fields: ResultField[]
   ├─ histories: ResultHistorySeries[]
   └─ summaries: ResultSummary[]
```

这里最需要读者记住的只有三点：

1. `fields` 只挂在 `frames` 下。
2. `histories` 和 `summaries` 直属于 `ResultStep`。
3. `ResultsSession` 是会话元数据，不是结果浏览门面。

### 4. 核心对象

#### 4.1 `ResultsWriter` 与 `ResultsReader`

##### `ResultsWriter`

`ResultsWriter` 是当前正式结果写出抽象接口。过程类执行时不会直接拼装 JSON 或内存结构，而是按这个接口逐步写入 `ResultsSession`、`ResultFrame`、`ResultHistorySeries` 和 `ResultSummary`。

###### 数据字段

当前抽象接口不冻结实例字段合同。具体字段由 `InMemoryResultsWriter`、`JsonResultsWriter` 等实现类持有。

###### 正式公开方法

`open_session(session)`  
返回 `None`。  
开启一个步骤结果写出会话。输入类型为 `ResultsSession`。

`close_session()`  
返回 `None`。  
关闭当前步骤会话。文件后端会在该阶段完成落盘。

`write_frame(frame)`  
返回 `None`。  
写入一个 `ResultFrame`。

`write_history_series(history)`  
返回 `None`。  
写入一个 `ResultHistorySeries`。

`write_history(history)`  
返回 `None`。  
这是兼容旧调用名的别名，内部仍调用 `write_history_series()`。

`write_summary(summary)`  
返回 `None`。  
写入一个 `ResultSummary`。

`get_capabilities()`  
返回 `ResultsCapabilities`。  
返回后端能力描述对象。

###### 当前可直接确认的行为

当前正式文件后端主线是 `JsonResultsWriter`，对应能力描述 `backend_name="json_reference"`、`is_reference_implementation=True`、`supports_append=False`、`supports_partial_storage_read=False`、`supports_restart_metadata=True`。

`InMemoryResultsWriter` 同时实现 `ResultsReader`，可用于测试、内存结果浏览和无文件写出的场景。

###### 当前边界

`ResultsWriter` 是抽象接口，不应写成某一种具体存储结构。

当前文档不将多后端生态扩写为既有正式能力；应以 `json_reference` 和 `memory_reference` 这两条已可确认实现为准。

###### 最小示例

```python
writer = JsonResultsWriter("outputs/demo.results.json")
writer.open_session(
    ResultsSession(
        model_name="beam-demo",
        step_name="step-static",
        procedure_type="static_linear",
    )
)
writer.write_frame(frame)
writer.close_session()
```

##### `ResultsReader`

`ResultsReader` 是当前正式结果读取抽象接口。`ResultsFacade`、`ResultsQueryService` 和 `ResultsProbeService` 都建立在这个接口之上。

###### 数据字段

当前抽象接口不冻结实例字段合同。

###### 正式公开方法

`read_database()`  
返回 `ResultsDatabase`。  
读取完整结果数据库。

`get_capabilities()`  
返回 `ResultsCapabilities`。  
读取结果后端能力。

`read_session()`  
返回 `ResultsSession`。  
读取结果会话元数据。

`list_steps()`  
返回 `tuple[str, ...]`。  
列出数据库中全部步骤名。

`read_step(step_name)`  
返回 `ResultStep`。  
读取单个步骤结果对象。

`read_frame(step_name, frame_id)`  
返回 `ResultFrame`。  
读取单个结果帧。

`read_field(step_name, frame_id, field_name)`  
返回 `ResultField`。  
读取单个结果场。

`read_history(step_name, history_name)`  
返回 `ResultHistorySeries`。  
读取单个历史量对象。

`read_summary(step_name, summary_name)`  
返回 `ResultSummary`。  
读取单个步骤摘要对象。

###### 当前边界

`ResultsReader` 不是探针接口，不直接返回 `ProbeSeries`。

#### 4.2 `ResultsSession` 与 `ResultsDatabase`

##### `ResultsDatabase`

`ResultsDatabase` 是当前正式结果数据库根对象。它持有一个 `ResultsSession` 和一个按步骤组织的 `ResultStep` 序列。

###### 数据字段

`session`  
类型为 `ResultsSession`。  
保存结果会话元数据。

`steps`  
类型为 `tuple[ResultStep, ...]`。  
按步骤顺序保存结果步对象。该元组保序。

`schema_version`  
类型为 `str`。  
保存结果 schema 版本。

`metadata`  
类型为 `Mapping[str, Any]`。  
保存数据库级元数据。

`ResultsSession.model_name`  
类型为 `str`。  
保存模型名称。

`ResultsSession.procedure_name`  
类型为 `str | None`。  
保存步骤过程名称。多步根会话可为空。

`ResultsSession.job_name`  
类型为 `str | None`。  
保存作业名称。

`ResultsSession.step_name`  
类型为 `str | None`。  
保存当前步骤名。多步根会话可为空。

`ResultsSession.procedure_type`  
类型为 `str | None`。  
保存步骤过程类型。多步根会话可为空。

`ResultsSession.metadata`  
类型为 `Mapping[str, Any]`。  
保存会话级附加信息。

`ResultsSession.database_id`  
类型为 `str`。  
当前默认通过 `uuid4().hex` 创建。

`ResultsSession.writer_backend`  
类型为 `str`。  
保存写出后端名称，例如 `json_reference`。

###### 正式公开方法

`is_multi_step`  
返回 `bool`。  
指示数据库是否包含多个正式步骤。

`frames`  
返回 `tuple[ResultFrame, ...]`。  
返回全部结果帧的扁平视图。

`histories`  
返回 `tuple[ResultHistorySeries, ...]`。  
返回全部历史量的扁平视图。

`summaries`  
返回 `tuple[ResultSummary, ...]`。  
返回全部摘要的扁平视图。

`list_steps()`  
返回 `tuple[str, ...]`。  
列出全部步骤名。

`get_step(step_name)`  
返回 `ResultStep`。  
按名称读取步骤结果对象。

`find_frame(step_name, frame_id)`  
返回 `ResultFrame`。  
按步骤名和帧号定位结果帧。

`to_dict()`  
返回 `dict[str, Any]`。  
将数据库对象序列化为字典。

`from_dict(payload)`  
返回 `ResultsDatabase`。  
从字典恢复数据库对象，并校验 schema major 兼容性。

###### 当前可直接确认的行为

`__post_init__()` 会校验步骤名不能重复。

当输入 payload 不含 `steps` 时，`from_dict()` 会转入 legacy payload 兼容路径。

多步写出时，根会话会把 `step_name`、`procedure_type`、`procedure_name` 置空，并在 `metadata["multi_step"]` 上标记多步语义。

###### 当前边界

`ResultsDatabase` 是逻辑结果对象，不是数据库引擎，也不是查询服务。

###### 最小示例

```python
database = ResultsDatabase(
    session=ResultsSession(
        model_name="beam-demo",
        step_name="step-static",
        procedure_type="static_linear",
    ),
    steps=(step,),
)
```

#### 4.3 `ResultStep`、`ResultFrame`、`ResultField`

##### `ResultStep`

`ResultStep` 是当前正式步骤结果对象。它把一个步骤中的帧、历史量和摘要组织在一起。

###### 数据字段

`name`  
类型为 `str`。  
保存步骤名。

`procedure_type`  
类型为 `str | None`。  
保存步骤过程类型，例如 `static_linear`、`modal`。

`step_index`  
类型为 `int | None`。  
保存步骤序号。

`frames`  
类型为 `tuple[ResultFrame, ...]`。  
保存该步骤全部结果帧。

`histories`  
类型为 `tuple[ResultHistorySeries, ...]`。  
保存该步骤全部历史量。

`summaries`  
类型为 `tuple[ResultSummary, ...]`。  
保存该步骤全部摘要。

`metadata`  
类型为 `Mapping[str, Any]`。  
保存步骤级附加元数据。

`restart_metadata`  
类型为 `Mapping[str, Any]`。  
保存与 restart 相关的附加元数据。

###### 正式公开方法

`get_frame(frame_id)`  
返回 `ResultFrame`。  
按帧号读取结果帧。

`get_field(frame_id, field_name)`  
返回 `ResultField`。  
按帧号和字段名读取结果场。

`get_history(history_name)`  
返回 `ResultHistorySeries`。  
按名称读取历史量。

`get_summary(summary_name)`  
返回 `ResultSummary`。  
按名称读取摘要。

`to_dict()`  
返回 `dict[str, Any]`。  
序列化步骤对象。

###### 当前可直接确认的行为

`__post_init__()` 会校验：每个 `ResultFrame.step_name` 必须等于当前步骤名；同一步骤内部 `frame_id` 不可重复；每个历史量和摘要对象也必须归属于当前步骤。

##### `ResultFrame`

`ResultFrame` 是当前正式结果帧对象。它表示某一个输出时刻、模态序号或其他轴位置上的一组结果场。

###### 数据字段

`frame_id`  
类型为 `int`。  
保存帧编号。

`step_name`  
类型为 `str`。  
保存所属步骤名。

`time`  
类型为 `float`。  
保存结果帧对应的时间值。模态帧仍保留该字段，但当前可为 `0.0`。

`fields`  
类型为 `tuple[ResultField, ...]`。  
保存该帧下全部结果场。

`metadata`  
类型为 `Mapping[str, Any]`。  
保存帧级附加元数据。

`frame_kind`  
类型为 `str`。  
当前常见值包括求解帧和模态帧标识。

`axis_kind`  
类型为 `str`。  
保存轴类型，例如 `TIME`、`FRAME_ID`、`MODE_INDEX`。

`axis_value`  
类型为 `float | int | None`。  
保存当前帧轴位置。若构造时未显式给出，当前实现会在 `__post_init__()` 中自动填充。

`restart_metadata`  
类型为 `Mapping[str, Any]`。  
保存 restart 相关帧级元数据。

###### 正式公开方法

`get_field(field_name)`  
返回 `ResultField`。  
按名称读取结果场对象。

`field_positions`  
返回 `tuple[str, ...]`。  
返回该帧中全部字段位置的去重序列。

`field_source_types`  
返回 `tuple[str, ...]`。  
返回该帧中全部字段来源类型的去重序列。

`target_keys`  
返回 `tuple[str, ...]`。  
返回该帧全部字段覆盖目标键的去重序列。

`target_count`  
返回 `int`。  
返回该帧覆盖目标数量。

`to_dict()`  
返回 `dict[str, Any]`。  
序列化结果帧对象。

###### 当前可直接确认的行为

若 `axis_value` 未显式给出，当前实现会在时间轴帧中使用 `time`，在其他轴类型中使用 `frame_id` 作为默认值。

##### `ResultField`

`ResultField` 是当前正式结果场对象。它描述一个变量在某一位置和某一目标集合上的值。

###### 数据字段

`name`  
类型为 `str`。  
保存正式字段名，例如 `U`、`RF`、`S`、`MODE_SHAPE`。

`position`  
类型为 `str`。  
保存结果位置，例如 `NODE`、`ELEMENT_CENTROID`、`INTEGRATION_POINT`、`ELEMENT_NODAL`、`NODE_AVERAGED`。构造时会被规范化。

`values`  
类型为 `Mapping[str, Any]`。  
保存 `target_key -> value` 映射。值可以是标量、元组、列表或分量字典。  
节点位移示例可写为 `{"part-1.n2": {"UX": 0.0012, "UY": -0.0048}}`。

`source_type`  
类型为 `str`。  
保存字段来源类型。当前常见值包括 `raw`、`recovered`、`averaged`、`derived`。

`component_names`  
类型为 `tuple[str, ...]`。  
保存分量名序列，例如 `("UX", "UY")` 或 `("S11", "S22", "S12")`。构造时若未显式给出，当前实现会尝试从 `values` 推断。

`target_keys`  
类型为 `tuple[str, ...]`。  
保存目标键序列。若构造时未显式给出，当前实现默认取 `values.keys()`。

`target_count`  
类型为 `int | None`。  
保存目标数量。若构造时未显式给出，当前实现取 `len(target_keys)`。

`metadata`  
类型为 `Mapping[str, Any]`。  
保存字段级附加信息，例如恢复元数据、平均分组元数据等。

###### 正式公开方法

`result_source`  
返回 `str`。  
这是 `source_type` 的只读别名属性。

`resolve_component_names(target_keys=None)`  
返回 `tuple[str, ...]`。  
当给定目标键子集时，该方法会基于子集值重新推断分量名；若无法推断，则返回当前对象的 `component_names`。

`to_dict()`  
返回 `dict[str, Any]`。  
序列化结果场对象。

###### 当前可直接确认的行为

`__post_init__()` 会规范化 `position` 和 `source_type`。

若 `values` 中出现不属于 `target_keys` 的键，构造过程会抛出错误。

若 `target_count` 小于 `target_keys` 的实际数量，构造过程会抛出错误。

###### 当前边界

当前帮助文档未将 `values` 中所有可能的数据外形冻结为单一格式。当前可直接确认的正式主线是“目标键映射”，而不是某一种固定数组布局。

#### 4.4 `ResultHistorySeries` 与 `ResultSummary`

- 所属层：结果层
- 家族：步骤结果家族
- 职责：一个表达“沿时间/模态等轴的历史量”，一个表达“步骤级汇总”
- 核心字段：`ResultHistorySeries` 以 `axis_kind / axis_values / values / paired_values` 为核心；`ResultSummary` 以 `name / data / metadata` 为核心
- 主要方法状态：当前以构造和字段访问为主
- 相邻对象关系：由 procedures 写出，并供 query、probe、summary 浏览接口读取
- 典型误解：history 不是 frame 列表，summary 也不是普通日志文本

`ResultHistorySeries` 当前最值得单独写清的是：

- `values` 是 `target_key -> 序列` 映射。
- `paired_values` 用于保存与主历史量配对的辅助序列。
- 序列长度必须与 `axis_values` 保持一致。

#### 4.5 最小结果对象示例

下面的片段不追求完整，而是让读者看到“一个步骤、一帧、一个字段”在结果层里的最小数据长相。

```python
frame = ResultFrame(
    frame_id=0,
    step_name="step-static",
    time=0.0,
    fields=(
        ResultField(
            name="U",
            position="NODE",
            values={
                "part-1.n1": {"UX": 0.0, "UY": 0.0},
                "part-1.n2": {"UX": 0.0012, "UY": -0.0048},
            },
            component_names=("UX", "UY"),
        ),
    ),
)

step = ResultStep(name="step-static", procedure_type="static_linear", frames=(frame,))
database = ResultsDatabase(
    session=ResultsSession(model_name="beam-demo", step_name="step-static", procedure_type="static_linear"),
    steps=(step,),
)
```

这个最小对象例子最值得读者记住的有四点：

1. `ResultField.values` 的键不是“随便一个标签”，而是正式 `target_key`，例如 `part-1.n2`。
2. 节点向量场的值通常长成“分量名字典”，例如 `{"UX": ..., "UY": ...}`。
3. `ResultFrame` 是同一输出时刻下多个字段的容器。
4. `ResultsDatabase` 才是整份结果对象图的根，不是 `ResultStep`。

如果换成积分点场，`values` 的键会变成类似 `part-1.e1.ip1` 的积分点键；如果换成单元质心场，键则更接近 `part-1.e1`。因此，结果层的数据外形和输出位置是直接相关的。

### 5. 结果访问接口

#### 5.1 `ResultsFacade`

`ResultsFacade` 是当前正式结果浏览门面类。它只依赖 `ResultsReader`，用于收口常见的结果读取入口。

##### 数据字段

`results_reader`  
类型为 `ResultsReader`。  
该字段保存底层结果读取器，例如 `JsonResultsReader` 或 `InMemoryResultsWriter`。

##### 正式公开方法

`session()`  
返回 `ResultsSession`。  
读取结果会话元数据。

`capabilities()`  
返回 `ResultsCapabilities`。  
读取结果后端能力。

`list_steps()`  
返回 `tuple[str, ...]`。  
列出全部步骤名。

`step(step_name)`  
返回 `ResultStep`。  
读取单个步骤对象。

`frame(step_name, frame_id)`  
返回 `ResultFrame`。  
读取单个结果帧。

`field(step_name, frame_id, field_name)`  
返回 `ResultField`。  
读取单个结果场对象。

`history(step_name, history_name)`  
返回 `ResultHistorySeries`。  
读取单个历史量对象。

`summary(step_name, summary_name)`  
返回 `ResultSummary`。  
读取单个步骤摘要对象。

`step_overviews(...)`  
返回 `tuple[ResultStepOverview, ...]`。  
按步骤名、过程类型、帧类型、轴类型、字段名、目标键和来源类型生成步骤概览。

`query()`  
返回 `ResultsQueryService`。  
创建查询服务对象。

`probe()`  
返回 `ResultsProbeService`。  
创建探针服务对象。

##### 当前可直接确认的行为

`step()`、`field()`、`history()`、`summary()` 当前都通过内部创建的 `ResultsQueryService` 完成读取。

##### 当前边界

`ResultsFacade` 不是 `ResultsDatabase` 本体，也不直接执行求解。

##### 最小示例

```python
results = ResultsFacade(JsonResultsReader("outputs/demo.results.json"))
step = results.step("step-static")
frame = results.frame("step-static", 0)
u_field = results.field("step-static", 0, "U")
```

#### 5.2 `ResultsQueryService`

`ResultsQueryService` 是当前正式结果查询服务类。它面向步骤、帧、字段、历史量和摘要的筛选与概览构造。

##### 数据字段

`results_reader`  
类型为 `ResultsReader`。  
该字段保存底层读取器。

##### 正式公开方法

`list_steps()`  
返回 `tuple[str, ...]`。  
列出全部步骤名。

`steps(procedure_type=None)`  
返回 `tuple[ResultStep, ...]`。  
按顺序读取步骤对象，并可按 `procedure_type` 过滤。

`step(step_name)`  
返回 `ResultStep`。  
读取单个步骤对象。

`frames(step_name, ..., field_name=None, frame_kind=None, axis_kind=None, frame_ids=None, source_type=None, position=None, target_key=None, result_key=None)`  
返回 `tuple[ResultFrame, ...]`。  
按步骤读取并筛选结果帧。若同时提供字段名、来源类型、位置或目标键，则当前实现会先筛选帧内字段，再保留至少包含一个匹配字段的结果帧。

`field(step_name, frame_id, field_name)`  
返回 `ResultField`。  
读取单个结果场。

`field_overview(step_name, frame_id, field_name, ..., target_key=None, result_key=None)`  
返回 `ResultFieldOverview`。  
读取单个结果场概览。

`field_overviews(step_name, ..., frame_id=None, field_name=None, frame_kind=None, axis_kind=None, source_type=None, position=None, target_key=None, result_key=None)`  
返回 `tuple[ResultFieldOverview, ...]`。  
按条件构造结果场概览集合。

`raw_field_overviews(step_name, **kwargs)`  
返回 `tuple[ResultFieldOverview, ...]`。  
筛选原始结果场概览。

`recovered_field_overviews(step_name, **kwargs)`  
返回 `tuple[ResultFieldOverview, ...]`。  
筛选恢复结果场概览。

`averaged_field_overviews(step_name, **kwargs)`  
返回 `tuple[ResultFieldOverview, ...]`。  
筛选平均结果场概览。

`derived_field_overviews(step_name, **kwargs)`  
返回 `tuple[ResultFieldOverview, ...]`。  
筛选派生结果场概览。

`history(step_name, history_name)`  
返回 `ResultHistorySeries`。  
读取单个历史量。

`histories(step_name, history_name=None, ..., axis_kind=None, paired_value_name=None)`  
返回 `tuple[ResultHistorySeries, ...]`。  
按名称、轴类型和配对值名称筛选历史量对象。

`summary(step_name, summary_name)`  
返回 `ResultSummary`。  
读取单个步骤摘要。

`summaries(step_name, summary_name=None, ..., data_key=None)`  
返回 `tuple[ResultSummary, ...]`。  
按摘要名或数据键筛选摘要对象。

##### 当前可直接确认的行为

当同时传入 `target_key` 或兼容别名 `result_key` 时，当前实现会把字段裁剪成只保留目标键对应的数据子集。

`field_overviews()` 当前会调用 `build_result_field_overview(...)` 计算最小值、最大值、目标数量和分量名等概览信息。

##### 当前边界

`ResultsQueryService` 返回的是正式结果对象或概览对象，不返回 `ProbeSeries`。

##### 最小示例

```python
query = results.query()
stress_frames = query.frames(
    "step-static",
    field_name="S",
    position="ELEMENT_CENTROID",
)
u_overviews = query.field_overviews("step-static", field_name="U")
```

#### 5.3 `ResultsProbeService`

`ResultsProbeService` 是当前正式探针服务类。它把结果对象中的历史量或字段分量投影为一维序列对象 `ProbeSeries`，并支持 CSV 导出。

##### 数据字段

`results_reader`  
类型为 `ResultsReader`。  
该字段保存底层读取器。

##### 正式公开方法

`history(step_name, history_name, target_key=None)`  
返回 `ProbeSeries`。  
把正式历史量对象投影为一条探针序列。结果序列会继承 `history.axis_kind`、`history.axis_values` 和历史量元数据。

`paired_history(step_name, history_name, value_name)`  
返回 `ProbeSeries`。  
从历史量对象的 `paired_values` 中抽取指定配对序列。

`field_component(step_name, target_key, component_name, *, field_name, frame_ids=None, expected_position=None)`  
返回 `ProbeSeries`。  
按目标键和分量名从一组帧中抽取单个字段分量序列。

`node_component(step_name, target_key, component_name, *, field_name="U", frame_ids=None)`  
返回 `ProbeSeries`。  
读取节点位置结果的单个分量序列。

`element_component(step_name, target_key, component_name, *, field_name, frame_ids=None)`  
返回 `ProbeSeries`。  
读取单元位置结果的单个分量序列。

`integration_point_component(step_name, target_key, component_name, *, field_name, frame_ids=None)`  
返回 `ProbeSeries`。  
读取积分点位置结果的单个分量序列。

`averaged_node_component(step_name, target_key, component_name, *, field_name, frame_ids=None)`  
返回 `ProbeSeries`。  
读取节点平均位置结果的单个分量序列。

`export_csv(probe_series, path)`  
返回 `Path`。  
把 `ProbeSeries` 导出为 CSV 文件。当前 CSV 表头可直接确认至少包括 `step_name`、`source_name`、`axis_kind`、`axis_value`、`value`、`field_name`、`target_key`、`resolved_target_key`、`component_name`、`position`、`source_type`、`metadata_json`。

##### 当前可直接确认的行为

`history()` 返回的 `ProbeSeries` 直接使用历史量对象的轴值，不会重新采样。

`field_component()` 在读取每个帧的字段后，会按目标键和分量名提取单个数值序列；若指定 `expected_position` 且字段位置不匹配，当前实现会抛出错误。

##### 当前边界

`ResultsProbeService` 不返回 `ResultField` 或 `ResultHistorySeries` 本体。

当前帮助文档未将 `ProbeSeries.metadata` 的全部键集合冻结为稳定公开合同。

##### 最小示例

```python
probe = results.probe()
tip_uy = probe.node_component("step-static", "part-1.n2", "UY", frame_ids=(0,))
time_series = probe.history("step-static", "TIME")
csv_path = probe.export_csv(tip_uy, "outputs/tip_uy.csv")
```

#### 5.4 最小读取与 probe 示例

```python
results = session.open_results(report.results_path)

step = results.step("step-static")
frame = results.frame("step-static", 0)
u_field = results.field("step-static", 0, "U")

tip_uy = results.probe().node_component("step-static", "part-1.n2", "UY", frame_ids=(0,))
time_history = results.probe().history("step-static", "TIME")
```

在这组调用中：

- `results.step(...)` 返回 `ResultStep`
- `results.frame(...)` 返回 `ResultFrame`
- `results.field(...)` 返回 `ResultField`
- `results.probe().node_component(...)` 返回 `ProbeSeries`

#### 5.5 `query` 与 `probe` 的区别示例

```python
query = results.query()
probe = results.probe()

stress_frames = query.frames("step-static", field_name="S", position="ELEMENT_CENTROID")
tip_curve = probe.node_component("step-static", "part-1.n2", "UY")
```

`query.frames(...)` 返回的是 `tuple[ResultFrame, ...]`。`probe.node_component(...)` 返回的是 `ProbeSeries`。两者面向的对象类型不同，不应混用。

### 6. 结果层与 procedures / solver 的边界

结果层与上游最容易混淆的边界如下：

- procedures 决定“什么时候写 frame/history/summary”，结果层决定“这些对象长什么样”。
- solver 持有 `RuntimeState`，结果层持有 `ResultFrame / ResultField / ResultHistorySeries`。
- `RuntimeState` 可以被 procedures 消费后生成结果，但它本身不是正式结果对象。

因此，帮助文档不应继续把“内部状态”和“正式结果”混写成同一个数据层。

### 7. 对象创建、引用与消费短例

```python
report = session.run_input_file("cases/beam.inp", step_name="Step-1")
results = session.open_results(report.results_path)
u = results.field("Step-1", 0, "U")
series = results.probe().node_component("Step-1", "part-1.n2", "UY")
```

这段短例对应四个连续动作：

1. `ProcedureRuntime` 在执行时通过 `ResultsWriter` 写出正式结果。
2. `session.open_results(...)` 通过 `ResultsReader` 打开 `ResultsDatabase`。
3. `ResultsFacade.field(...)` 读取的是正式 `ResultField`。
4. `ResultsProbeService.node_component(...)` 消费的是结果对象，而不是直接读取 solver 内部状态。

### 8. 边界对照表

下表用于澄清本层最容易混淆的相邻对象或相邻层。

| 对象或边界对 | 所属层 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- | --- |
| `RuntimeState` vs `ResultFrame` | solver / 结果层 | 一个是内部求解状态，一个是正式帧结果 | 不是内存版与磁盘版同一对象 | 都可能包含位移语义 |
| `ResultsSession` vs `ResultsDatabase` | 结果层 | 一个是会话元数据，一个是完整结果层级 | 不是同一个对象的两个名字 | 都在数据库根部出现 |
| `ResultsDatabase` vs `ResultsFacade` | 结果层 / 消费侧 | 一个是结果本体，一个是浏览门面 | 不是数据库与 reader 的同义词 | facade 经常被误写成数据库 |
| `ResultsQueryService` vs `ResultsProbeService` | 结果层消费侧 | 一个做筛选与概览，一个做序列抽取 | 不是相同服务的不同别名 | 都由 facade 派生 |
| 结果层 vs VTK exporter | 结果层 / 产品壳层 | 一个定义结果协议，一个做外部导出 | 不是同层后端对象 | exporter 消费结果，但不定义结果层级 |

### 9. 本章主线图

```text
ProcedureRuntime
    -> ResultsWriter
    -> ResultsSession / ResultsDatabase
    -> ResultsReader
    -> ResultsFacade
    -> ResultsQueryService / ResultsProbeService / exporter / GUI
```

`JsonResultsReader` 当前能力描述包括：

- `backend_name="json_reference"`
- `is_reference_implementation=True`
- `supports_append=False`
- `supports_partial_storage_read=False`
- `supports_restart_metadata=True`

它说明读端和写端的能力边界是一致的。

#### 3.6 Legacy 兼容

`ResultsDatabase.from_dict()` 当前会在发现 payload 中没有 `steps` 时回退到 legacy 读取路径。

因此，正式文档可以写成：

- 当前结果格式主线是分层的 `steps` 结构
- 旧版扁平 payload 仍有兼容读取逻辑

但帮助文档正文应优先讲当前的分层结构，而不是再以 legacy 结构做主叙述。

### 4. 内存结果后端

当前还存在一个正式内置但更偏脚本 / 测试用途的结果后端：

- `memory`

对应实现是：

- `InMemoryResultsWriter`

它的能力描述包括：

- `backend_name="memory_reference"`
- `is_reference_implementation=True`
- `supports_append=False`
- `supports_partial_storage_read=False`
- `supports_restart_metadata=True`

因此：

- `memory` 适合测试代码、即时脚本和中间处理
- `json` 仍然是当前帮助文档中最稳妥的文件型主线

### 5. 多步结果结构

#### 5.1 根会话

当同一个 writer 连续接收多个步骤结果时，会生成多步根会话。当前根会话的特征包括：

- `metadata["multi_step"] = True`
- `procedure_name = None`
- `step_name = None`
- `procedure_type = None`

这意味着多步根会话明确与单步步骤会话区分开来。

#### 5.2 元数据裁剪

多步根会话在构造时，会从原始根会话元数据中去掉：

- `step_parameters`
- `output_request_names`

这样做的含义是：

- 多步根会话只保留跨步骤共享的上层元信息
- 具体步骤参数仍属于各自 `ResultStep` 的局部语义

#### 5.3 文档建议

帮助文档中应把多步结果写成：

- 一个数据库
- 多个步骤
- 一个多步根会话

而不要再把它写成旧式“一个文件只有一个步骤”的固定格式。

### 6. 结果字段体系

#### 6.1 结果位置

当前正式结果位置包括：

- `NODE`
- `ELEMENT_CENTROID`
- `INTEGRATION_POINT`
- `ELEMENT_NODAL`
- `NODE_AVERAGED`
- `GLOBAL_HISTORY`

这组位置与第 8 章、第 9 章中的输出请求位置完全对应。

#### 6.2 结果来源类型

当前正式结果来源类型包括：

- `raw`
- `recovered`
- `averaged`
- `derived`

它们的含义可以概括为：

- `raw`：直接由过程或单元输出产生
- `recovered`：由积分点场恢复到单元节点
- `averaged`：由恢复场再平均到节点
- `derived`：由已有场进一步计算得到

#### 6.3 场构建链路

`StepProcedureRuntime` 当前会按以下顺序构建字段：

1. 先放入节点向量场，例如 `U`、`RF`、`MODE_SHAPE`
2. 再从单元输出构建原始单元场
3. 再构建 recovered 字段
4. 再构建 averaged 字段
5. 最后构建 derived 字段

这说明字段不是平铺并列生成，而是存在明确依赖链。

### 7. 原始字段

#### 7.1 节点向量类原始字段

当前正式节点向量类原始字段包括：

- `U`
- `RF`
- `MODE_SHAPE`

它们位于：

- `NODE`

#### 7.2 单元质心原始字段

当前正式单元质心字段包括：

- `S`
- `E`
- `SECTION`

它们位于：

- `ELEMENT_CENTROID`

其中 `SECTION` 不是应力或应变，而是单元输出中与截面相关的结构化字段。

#### 7.3 积分点原始字段

当前正式积分点字段包括：

- `S_IP`
- `E_IP`

它们位于：

- `INTEGRATION_POINT`

#### 7.4 历史量原始字段

当前正式全局历史变量包括：

- `TIME`
- `FREQUENCY`

它们位于：

- `GLOBAL_HISTORY`

### 8. 恢复字段与平均字段

#### 8.1 Recovered 字段

恢复服务当前会从积分点应力、应变构造：

- `E_REC`
- `S_REC`

它们位于：

- `ELEMENT_NODAL`

恢复过程中还会写入丰富元数据，例如：

- owner element
- base target key
- averaging domain
- averaging weight
- measure metadata

这也是为什么后续平均和 probe 能继续追踪来源。

#### 8.2 Recovery 策略边界

`RecoveryService.available_strategies()` 会声明三种策略名：

- `direct_extrapolation`
- `least_squares`
- `patch`

但当前真正实现的只有：

- `direct_extrapolation`

如果传入其他策略，服务会直接抛出 `NotImplementedError`。

因此，帮助文档必须避免把 `least_squares` 和 `patch` 写成已经可正式使用的恢复方法。

#### 8.3 Averaged 字段

当前平均服务只从恢复应力场构造：

- `S_AVG`

它位于：

- `NODE_AVERAGED`

这条限制非常重要，因为它说明“节点平均”当前正式主线是围绕应力组织的，而不是任意字段都可平均。

#### 8.4 关于 `E_AVG`

当前 GUI 层和展示层能看到 `E_AVG` 相关名字，但在正式求解输出规划器和字段构建链路中：

- 没有 `E_AVG` 的正式输出主线

因此，帮助文档应明确写成：

- 当前正式平均字段是 `S_AVG`
- 不应把 `E_AVG` 写成求解器已经正式输出的字段

### 9. 派生字段

#### 9.1 位移模

派生字段服务会从 `U` 构造：

- `U_MAG`

它的来源类型是：

- `derived`

#### 9.2 等效应力与主应力

派生字段服务会从三类应力字段构造派生字段：

- 从 `S_IP` 构造 `S_VM_IP` 与 `S_PRINCIPAL_IP`
- 从 `S_REC` 构造 `S_VM_REC` 与 `S_PRINCIPAL_REC`
- 从 `S_AVG` 构造 `S_VM_AVG` 与 `S_PRINCIPAL_AVG`

这些字段都属于：

- `derived`

#### 9.3 当前边界

当前正式派生字段主线应理解为：

- `U_MAG`
- `S_VM_*`
- `S_PRINCIPAL_*`

帮助文档不应把 GUI 中出现的其他家族名、简写名或中心值名变体，自动写成正式输出字段。

### 10. 分量体系

#### 10.1 应变分量

当前正式应变分量名按单元类型区分：

- `B21`：`E11`
- `CPS4`：`E11`、`E22`、`E12`
- `C3D8`：`E11`、`E22`、`E33`、`E12`、`E23`、`E13`

#### 10.2 应力分量

当前正式应力分量名按单元类型区分：

- `B21`：`S11`
- `CPS4`：`S11`、`S22`、`S12`
- `C3D8`：`S11`、`S22`、`S33`、`S12`、`S23`、`S13`

#### 10.3 主应力分量

当前主应力分量统一使用：

- `P1`
- `P2`
- `P3`

即使在二维问题中，也仍沿用这一套正式命名。

### 11. 查询与结果浏览

#### 11.1 结果门面类的方法状态表

| 类 | 方法状态 | 说明 |
| --- | --- | --- |
| `ResultsFacade` | 门面类 | 负责把 reader-only 结果浏览收口为统一入口，兼顾直接访问、概览浏览和下钻到 query / probe |
| `ResultsQueryService` | 服务类 | 负责按步骤、帧、字段、历史量和摘要做批量筛选与概览提取 |
| `ResultsProbeService` | 服务类 | 负责把历史量或字段分量投影为 probe 序列，并提供 CSV 导出 |

#### 11.2 `ResultsFacade` 方法合同表

所属层：结果层，reader-only 门面。

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 | 常见误解 |
| --- | --- | --- | --- | --- | --- | --- |
| `session()` | 无 | `ResultsSession` | 读取当前结果会话元数据 | 想先看结果文件属于哪个模型、作业和步骤上下文时 | 高级脚本、GUI 结果上下文 | 它返回的是会话元数据，不是步骤结果本体 |
| `capabilities()` | 无 | `ResultsCapabilities` | 读取当前 reader 的能力边界 | 需要确认后端支持哪些读取能力时 | 高级脚本、维护者、GUI | capability 不是功能自动可用的保证，只是后端声明 |
| `list_steps()` | 无 | `tuple[str, ...]` | 列出结果中的正式步骤名 | 想知道结果里有哪些步骤时 | 普通用户、高级脚本、GUI | 它只返回步骤名，不返回步骤对象详情 |
| `step()` / `frame()` / `field()` / `history()` / `summary()` | 按路径提供 `step_name`、`frame_id`、`field_name`、`history_name`、`summary_name` | 单个 `ResultStep` / `ResultFrame` / `ResultField` / `ResultHistorySeries` / `ResultSummary` | 直接读取单个正式结果对象 | 已知访问路径，想直取结果对象时 | 普通用户、高级脚本、GUI | 这些方法走的是 reader-only 读取，不会修改结果数据库 |
| `step_overviews()` | 可选步骤、过程类型、帧/字段/目标筛选条件 | `tuple[ResultStepOverview, ...]` | 生成适合浏览器或摘要面板消费的步骤概览 | 需要按条件快速浏览多个步骤时 | GUI、高级脚本 | 它返回的是概览对象，不是完整 `ResultStep` |
| `frames()` | 可选步骤、帧类型、轴类型、字段、目标、来源类型、位置等筛选条件 | `tuple[ResultFrameOverview, ...]` | 浏览帧级概览 | 需要先按条件缩小帧集合时 | GUI、高级脚本 | 返回值不是原始 `ResultFrame`，而是概览视图 |
| `fields()` | 可选步骤、帧、字段、来源类型、位置、目标筛选条件 | `tuple[ResultFieldOverview, ...]` | 浏览字段级概览 | 需要按来源或位置筛选字段时 | GUI、高级脚本 | 它不直接返回字段值映射，而是概览摘要 |
| `raw_fields()` / `recovered_fields()` / `averaged_fields()` / `derived_fields()` | 与 `fields()` 相同 | `tuple[ResultFieldOverview, ...]` | 按 `source_type` 预设筛选字段概览 | 想直接看原始、恢复、平均或派生字段时 | GUI、高级脚本 | 这些是 `fields(source_type=...)` 的便捷入口，不是独立结果层 |
| `histories()` / `summaries()` | 可选步骤、名称和筛选条件 | `tuple[ResultHistoryOverview, ...]` / `tuple[ResultSummaryOverview, ...]` | 浏览历史量或摘要概览 | 想批量看历史量和摘要，而不是取单条对象时 | GUI、高级脚本 | 返回的是概览，不是 `ResultHistorySeries` / `ResultSummary` 本体 |
| `query()` | 无 | `ResultsQueryService` | 下钻到批量查询服务 | 需要做更细粒度筛选时 | 高级脚本、GUI | `query()` 不会预先缓存或物化整库，只是创建 reader-only 服务 |
| `probe()` | 无 | `ResultsProbeService` | 下钻到 probe 服务 | 想把结果抽成一条时序或分量序列时 | 普通用户、高级脚本、GUI | `probe()` 不等于任意字段都能自动猜出分量和目标键 |

#### 11.3 `ResultsQueryService` 方法合同表

所属层：结果层，reader-only 查询服务。

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 | 常见误解 |
| --- | --- | --- | --- | --- | --- | --- |
| `list_steps()` | 无 | `tuple[str, ...]` | 列出全部步骤名 | 想先看查询范围时 | 高级脚本、GUI | 与 `ResultsFacade.list_steps()` 语义一致，不是不同结果源 |
| `steps()` | 可选 `procedure_type` | `tuple[ResultStep, ...]` | 读取并可按过程类型筛选步骤对象 | 需要拿到步骤对象本体时 | 高级脚本、GUI | 这不是步骤概览；返回的是完整 `ResultStep` |
| `step()` | `step_name` | `ResultStep` | 读取单个步骤对象 | 已知步骤名时 | 高级脚本、GUI | 它不会自动容错到“最近一步” |
| `frames()` | `step_name` 与可选字段、帧类型、轴类型、`frame_ids`、来源类型、位置、目标筛选条件 | `tuple[ResultFrame, ...]` | 按条件读取一批正式帧对象 | 需要先拿一组帧再自己处理时 | 高级脚本、GUI | 返回的是完整帧，不是概览 |
| `field()` | `step_name`、`frame_id`、`field_name` | `ResultField` | 读取单个字段对象 | 已知字段路径时 | 普通用户、高级脚本、GUI | 它不做 probe 式分量抽取 |
| `field_overview()` | `step_name`、`frame_id`、`field_name`，可选目标筛选条件 | `ResultFieldOverview` | 读取单个字段概览 | 只想看字段元数据摘要时 | 高级脚本、GUI | 如果没有匹配字段会显式报错，不会返回空占位对象 |
| `field_overviews()` | `step_name` 与可选帧、字段、来源、位置、目标筛选条件 | `tuple[ResultFieldOverview, ...]` | 批量生成字段概览 | 做字段浏览面板或批量筛选时 | GUI、高级脚本 | 概览对象不包含完整 `values` 映射 |
| `raw_field_overviews()` / `recovered_field_overviews()` / `averaged_field_overviews()` / `derived_field_overviews()` | `step_name` 与可选筛选参数 | `tuple[ResultFieldOverview, ...]` | 对 `field_overviews()` 做按 `source_type` 的便捷筛选 | 想快速查看某一字段来源族时 | GUI、高级脚本 | 这些是便捷包装，不是新的底层结果类型 |
| `history()` / `histories()` | `step_name`，可选名称、轴类型、`paired_value_name` | 单个或多个 `ResultHistorySeries` | 读取历史量对象 | 想拿正式历史量对象或批量筛选历史量时 | 高级脚本、GUI | 它与 probe `history()` 不同；这里返回的是正式结果对象，不是 `ProbeSeries` |
| `summary()` / `summaries()` | `step_name`，可选名称、`data_key` | 单个或多个 `ResultSummary` | 读取步骤摘要对象 | 需要直接访问摘要数据时 | 高级脚本、GUI | 摘要不是 history，也不是 frame 上的字段 |

#### 11.4 使用边界与文档口径

帮助文档中应把三层入口区分为：

- `ResultsFacade`：面向普通用户和 GUI 的统一浏览入口。
- `ResultsQueryService`：面向批量筛选与概览生成。
- `ResultsProbeService`：面向“把结果抽成一条序列”的探针服务。

因此，不应把：

- `ResultsFacade.field()` 与 `ResultsProbeService.node_component()` 混写成同一种读取方式；
- `ResultsQueryService.field_overviews()` 与 `ResultField` 本体混写；
- `ResultsFacade.query()` / `probe()` 混写成结果数据库对象本身。

### 12. Probe 与 CSV 导出

#### 12.1 `ResultsProbeService` 方法状态表

所属层：结果层，reader-only probe 服务。

| 类 | 方法状态 | 说明 |
| --- | --- | --- |
| `ResultsProbeService` | 服务类 | 当前公开方法以历史量 probe、字段分量 probe 和 CSV 导出为主；内部筛帧、分量抽取与目标解析 helper 不应写成公共接口 |

#### 12.2 `ResultsProbeService` 方法合同表

| 方法 | 输入 | 返回 | 作用 | 何时调用 | 谁会调用 | 常见误解 |
| --- | --- | --- | --- | --- | --- | --- |
| `history()` | `step_name`、`history_name`，可选 `target_key` | `ProbeSeries` | 把正式历史量对象投影成一条 probe 序列 | 已有历史量，想直接画曲线或导出时 | 普通用户、高级脚本、GUI | 返回的不是 `ResultHistorySeries`，而是 probe 序列 |
| `paired_history()` | `step_name`、`history_name`、`value_name` | `ProbeSeries` | 从历史量的 `paired_values` 中抽出一条配对序列 | 历史量同时保存多类配对值时 | 高级脚本、GUI | 它不是任意字段对的自动组合，只能读取历史量里已有的 paired value |
| `field_component()` | `step_name`、`target_key`、可选 `component_name`，以及 `field_name`、可选 `frame_ids`、`expected_position` | `ProbeSeries` | 通用字段分量 probe 入口 | 想从一组帧里抽某字段某目标的分量序列时 | 高级脚本、GUI | 它要求调用方明确 `field_name`，并在有歧义时明确 `component_name` |
| `node_component()` | `step_name`、`node_key`、可选 `component_name`、可选 `field_name`、`frame_ids` | `ProbeSeries` | 节点分量 probe 便捷入口 | 提取位移、振型等节点量时 | 普通用户、高级脚本、GUI | 默认字段通常是位移族；它不是任意位置字段的万能入口 |
| `element_component()` | `step_name`、`element_key`、可选 `component_name`、可选 `field_name`、`frame_ids` | `ProbeSeries` | 单元质心分量 probe 便捷入口 | 提取单元应力等单元量时 | 普通用户、高级脚本、GUI | 它期望字段位置与单元质心语义一致 |
| `integration_point_component()` | `step_name`、`integration_point_key`、可选 `component_name`、可选 `field_name`、`frame_ids` | `ProbeSeries` | 积分点分量 probe 便捷入口 | 提取积分点结果时 | 高级脚本、GUI | 如果字段位置不是积分点，接口会显式拒绝 |
| `averaged_node_component()` | `step_name`、`node_key`、可选 `component_name`、可选 `field_name`、`frame_ids` | `ProbeSeries` | 节点平均场分量 probe 便捷入口 | 提取平均节点应力等平均量时 | 普通用户、高级脚本、GUI | 基础节点键与平均目标键不一定相同，接口可能做解析或要求更具体键 |
| `export_csv()` | `probe_series`、`path` | `Path` | 把 probe 序列导出为 CSV | 需要外部复盘、作图或归档时 | 普通用户、高级脚本、GUI | 它导出的是 probe 结果，不是整库字段导出器 |

#### 12.3 Probe 的定位

可以把几类 probe 简单理解为：

- `history()` / `paired_history()`：把正式历史量改写成可直接消费的一维序列。
- `field_component()`：通用字段分量抽取器。
- `node_component()` / `element_component()` / `integration_point_component()` / `averaged_node_component()`：按结果位置预设好的便捷入口。

因此，probe 更像“从结果对象中投影出一条序列”，而不是重新定义一套新的结果数据库层级。

#### 12.4 Probe 限制

当前 probe 服务在两类歧义场景下会显式报错：

- 目标字段有多个分量，但调用方没有给出 `component_name`。
- 节点平均量无法从基础节点键唯一解析到当前平均目标键。

此外，还应明确以下边界：

- 如果字段位置与 probe 入口预期位置不一致，接口会 fail-fast，而不是自动改道到别的位置。
- `field_component()` 读取的是正式 `ResultField`，因此 `field_name`、`target_key` 和 `frame_ids` 必须能在正式结果层解析成功。

这意味着 probe 接口是“显式而严格”的，而不是自动猜测的接口。

#### 12.5 CSV 导出

`export_csv()` 当前写出的表头包括：

- `step_name`
- `source_name`
- `axis_kind`
- `axis_value`
- `value`
- `field_name`
- `target_key`
- `resolved_target_key`
- `component_name`
- `position`
- `source_type`
- `metadata_json`

因此，probe CSV 不是仅供画一条曲线的最简格式，而是保留了足够多的来源信息，方便进一步复盘。

### 13. VTK 导出

#### 13.1 功能定位

当前正式外部导出格式是：

- `vtk`

对应导出器是：

- `VtkExporter`

它导出的是：

- VTK legacy 文本

#### 13.2 默认选择规则

如果 `export()` 调用时没有给出：

- `step_name`
- `frame_id`

导出器会默认选择：

- 最后一个结果步骤
- 该步骤中的最后一帧

因此，对普通用户来说，“导出当前最新结果”是默认行为。

#### 13.3 文件结构

当前导出器写出的头部结构包括：

- `# vtk DataFile Version 3.0`
- `ASCII`
- `DATASET UNSTRUCTURED_GRID`

这说明当前 VTK 导出主线是 legacy ASCII unstructured grid，而不是 XML 系列格式。

#### 13.4 几何组织

导出器会从 `model.iter_compilation_scopes()` 收集网格：

- 节点被组织为 `POINTS`
- 单元连接被组织为 `CELLS`
- 单元类型被组织为 `CELL_TYPES`

二维坐标会补零扩展到三维坐标，这样才能进入统一的 VTK 点坐标格式。

#### 13.5 字段到 VTK 数据区的映射

当前字段映射规则可以概括为：

- `NODE` -> `POINT_DATA`
- `NODE_AVERAGED` -> `POINT_DATA`
- `ELEMENT_CENTROID` -> `CELL_DATA`
- `INTEGRATION_POINT` -> `CELL_DATA`
- `ELEMENT_NODAL` -> `CELL_DATA`

其中：

- 积分点和单元节点场会按 slot 形式展开为多个单元数组
- 全局历史量与摘要不会进入 VTK 场数组

#### 13.6 向量与标量

当前只有以下节点场会作为 VTK 向量场写出：

- `U`
- `MODE_SHAPE`

其他数值型字段通常会按分量拆成标量数组。

数组名会带来源前缀，例如：

- `RAW`
- `RECOVERED`
- `AVERAGED`
- `DERIVED`

因此，VTK 文件中的数组名本身就保留了结果来源语义。

#### 13.7 单元类型映射

当前正式 VTK cell type 映射是：

- `B21 -> 3`
- `CPS4 -> 9`
- `C3D8 -> 12`

如果遇到其他单元类型，导出器会直接报错，而不会尝试猜测映射。

#### 13.8 当前限制

使用 VTK 导出时应明确以下边界：

- 只支持当前三种正式单元的映射。
- 非数值分量不会成为标准 VTK 数组。
- 全局历史量和摘要不会被导出到 VTK。
- 帮助文档不应把 VTK 导出写成“完整结果数据库导出”，它更准确地说是“单帧场结果导出”。

### 14. 使用建议

- 需要稳定文件结果时，优先使用 `json` 后端。
- 需要测试或脚本内中转时，可使用 `memory` 后端。
- 说明字段时，优先从“位置 + 来源类型 + 变量名”三个维度写清楚。
- 对平均和派生字段，务必区分正式主线与 GUI 变体名，不要把 `E_AVG` 等展示层词汇当成正式输出承诺。
- 需要外部可视化时，优先走 `json -> ResultsFacade / export_results() -> vtk` 这条正式链路。

## 第 13 章 作业、快照与派生算例

本章说明 pyFEM 当前正式主线中的“如何运行一个模型”。重点不是按钮位置本身，而是把输入文件、内存模型、快照、结果文件、导出文件、运行报告和 GUI 工作台之间的关系写清楚。

### 1. 功能概览

当前正式作业链路由三层组成：

- `JobManager`：负责统一的加载、编译、执行、结果写出与结果导出。
- `JobSnapshotService`：负责把当前模型冻结成可复跑的 `snapshot`，并在复跑后回写 manifest。
- `GuiShell`：负责 GUI 场景下的标准运行入口、最近一次快照状态、外进程求解请求和结果路径状态。

把它们连起来后，可以得到当前正式主线：

1. 载入 `INP` 或准备一个 `ModelDB`。
2. 解析这次到底运行哪个步骤。
3. 编译为 `CompiledModel`。
4. 选择结果后端并运行正式 `ProcedureRuntime`。
5. 生成 `results`。
6. 如有需要，再通过正式 exporter 导出 `vtk`。
7. 返回 `JobExecutionReport`。
8. 若走 GUI 运行路径，则优先冻结 `snapshot`，再基于冻结文件复跑。

这意味着：

- 命令行/脚本主线的核心对象是 `JobManager`。
- GUI 主线不是直接“拿活模型求解”，而是先冻结为 `snapshot` 再运行。
- `snapshot`、`manifest`、`results`、`report` 是一组关联工件，而不是互相独立的临时文件。

### 2. 标准作业执行

#### 2.1 `JobExecutionRequest`

一次标准作业请求由 `JobExecutionRequest` 描述。它最重要的合同有两条：

- 必须且只能提供 `input_path` 或 `model` 其中之一。
- 只有在给出 `export_format` 时，才允许同时提供 `export_path`。

这两个约束都是 fail-fast 约束。作业在进入编译和求解阶段之前，就会先拦截“来源不明确”和“导出参数不完整”这两类错误。

当前请求对象里最关键的字段包括：

| 字段 | 作用 |
| --- | --- |
| `input_path` | 从输入文件运行 |
| `model` | 直接从内存中的 `ModelDB` 运行 |
| `model_name` | 给 importer 场景补充模型名 |
| `importer_key` | 当前默认正式 importer 为 `inp` |
| `step_name` | 显式指定本次运行的步骤名 |
| `results_backend` | 当前正式后端主要是 `json` 与 `memory` |
| `results_path` | 结果路径；对 `json` 后端尤其重要 |
| `export_format` | 当前正式外部导出格式是 `vtk` |
| `export_path` | 导出路径 |

这些字段里最容易被误解的是后三项：

- `results_backend="json"` 表示结果会进入正式结果文件主线；这时 `results_path` 决定结果文件位置。
- `results_backend="memory"` 更适合测试或内部中转；它不是“自动生成一个隐藏文件结果”的意思。
- `export_format="vtk"` 只表示“在正式结果写出之后，再走一次 exporter”；它不会替代 `.results.json` 主结果。

最小请求对象通常长成下面两类之一：

```python
# 例 1：从输入文件运行
request = JobExecutionRequest(
    input_path="cases/beam.inp",
    step_name="LOAD_STEP",
    results_backend="json",
    results_path="cases/beam.results.json",
)
```

```python
# 例 2：从内存模型运行，并显式给出落盘路径
request = JobExecutionRequest(
    model=model,
    step_name="step-static",
    results_backend="json",
    results_path="out/step-static.results.json",
    export_format="vtk",
    export_path="out/step-static.vtk",
)
```

这两个例子分别对应本章后面会继续展开的两条正式入口：

- `run_input_file()` 适合“已经有一份正式输入文件，直接运行并落盘结果”。
- `run_model()` 或 `execute(JobExecutionRequest(...))` 适合“模型已经在内存里，继续求解或导出”。

#### 2.2 `JobManager.execute()` 的统一流程

`JobManager.execute()` 是标准作业主线的统一外壳。它内部固定按以下顺序工作：

1. `_load_model()`：从 `input_path` 导入模型，或者直接接收 `ModelDB`。
2. `_resolve_results_path()`：解析本次结果写出路径。
3. `_resolve_export_path()`：解析本次导出路径。
4. `Compiler(...).compile(model)`：编译模型。
5. `resolve_step_name(model, request.step_name)`：决定本次正式执行的步骤。
6. `_create_results_writer()`：创建结果写出器。
7. `compiled_model.get_step_runtime(...).run(writer)`：执行正式步骤运行时。
8. `_resolve_results_reader()`：为 exporter 构造正式 `ResultsReader`。
9. 如给出 `export_format`，则调用正式 exporter。
10. 汇总帧数、历史量数、摘要数，返回 `JobExecutionReport`。

因此，帮助文档里不应把“运行”“写结果”“导出 VTK”写成三条互不相干的链路；它们在正式实现中本来就是同一条主线上的不同阶段。

#### 2.3 输入文件执行

输入文件路径对应 `run_input_file()` 这条便捷入口。它本质上仍然是构造一个 `JobExecutionRequest` 并交给 `execute()`。

在 `input_path` 场景下，默认行为最清晰：

- 如果结果后端是 `json` 且没有显式给出 `results_path`，系统会按输入文件路径生成默认结果文件路径。
- 如果要求导出 `vtk` 且没有显式给出 `export_path`，系统也会按输入文件路径生成默认导出路径。
- 这使得“给一份 `INP` 直接运行并落盘结果”成为当前最完整、最省心的正式入口。

如果输入文件不存在，系统会直接抛出：

- `输入文件不存在: ...`

如果 `step_name` 未指定，则会继续进入步骤解析规则；详见本章第 2.5 节。

#### 2.4 `ModelDB` 执行

内存模型对应 `run_model()` 入口。它适合：

- 测试中构造 benchmark 模型后直接运行。
- GUI 内部在冻结之前操作活模型。
- 脚本场景下自行组装 `ModelDB` 后求解。

但 `ModelDB` 直跑与输入文件直跑有一个明显差异：

- 如果结果后端选择 `json`，又没有给 `results_path`，系统无法从 `input_path` 推导默认路径，因此会 fail-fast。
- 如果要求导出，但没有给 `export_path`，同样无法自动推导默认导出路径。

所以在 `ModelDB` 场景下，通常有两种正式用法：

- 选择 `memory` 结果后端，在内存内消费结果。
- 继续使用 `json`，但显式提供 `results_path`；如还要导出，再显式提供 `export_path`。

换句话说，`ModelDB` 运行不是不能落盘，而是必须把路径说清楚。

#### 2.5 步骤选择规则

本次到底运行哪个步骤，由 `resolve_step_name()` 统一决定。当前规则是：

1. 如果显式给了 `step_name`，则必须真有这个步骤。
2. 如果没有显式给 `step_name`，而 `job.step_names` 中只有一个步骤，就使用它。
3. 如果 `job.step_names` 有多个步骤，则必须显式指定。
4. 如果模型本身只有一个步骤，就直接运行这一个。
5. 如果模型没有步骤，或模型中有多个步骤但没有指定，则直接报错。

因此，下面几类报错都属于“步骤选择阶段”而不是“求解器异常”：

- `模型中不存在步骤 ...`
- `当前作业包含多个步骤，请显式指定 step_name。`
- `当前模型中没有可运行的步骤。`
- `当前模型包含多个步骤，请显式指定 step_name。`

#### 2.6 最小运行示例

下面三段最小例子分别对应当前最常见的三种正式作业入口。代码块只强调对象关系和参数语义，省略导入路径。

```python
# 例 1：从输入文件直接运行
session = PyFEMSession()
report = session.run_input_file(
    "cases/beam.inp",
    step_name="LOAD_STEP",
    results_path="cases/beam.results.json",
)

print(report.step_name)
print(report.results_path)
print(report.frame_count)
```

这条路径最适合“手里已经有一份 `INP`，想最快得到正式结果文件”的场景。`PyFEMSession` 会把请求转交给 `JobManager`，返回的 `report` 则告诉你这次实际运行了哪个步骤、结果落在什么位置、生成了多少帧和历史量。

```python
# 例 2：从内存中的 ModelDB 运行
session = PyFEMSession()
report = session.run_model(
    model,
    step_name="step-static",
    results_backend="json",
    results_path="out/model.results.json",
)

print(report.procedure_type)
print(report.results_path)
```

这条路径最适合测试、脚本建模或 GUI 内部活模型场景。关键点不是“能不能跑”，而是“路径必须说清楚”。一旦结果要落盘，`results_path` 就不应再省略。

```python
# 例 3：显式使用 JobManager 和 monitor
monitor = InMemoryJobMonitor()
manager = JobManager()
report = manager.execute(
    JobExecutionRequest(
        input_path="cases/beam.inp",
        step_name="LOAD_STEP",
        results_path="cases/beam.results.json",
        export_format="vtk",
        export_path="cases/beam.vtk",
    ),
    monitor=monitor,
)

print(monitor.snapshot())
print(report.export_path)
```

这条路径最适合你需要显式控制请求对象、监控消息或导出参数的场景。这里的 `monitor.snapshot()` 收集到的是作业阶段消息，例如“加载模型”“编译模型”“执行步骤”“导出结果 vtk”这类正式主线消息。

#### 2.7 `JobExecutionReport` 的数据长相与使用方式

`JobExecutionReport` 不是给 GUI 专用的临时摘要，而是标准作业返回给上层壳层的正式结果对象。它当前至少包含以下字段：

| 字段 | 数据形式 | 运行后如何理解 |
| --- | --- | --- |
| `model_name` | `str` | 这次实际执行的是哪个模型 |
| `job_name` | `str | None` | 若模型定义了正式 `JobDef`，这里可回看作业名 |
| `step_name` | `str` | 本次真正执行的步骤名 |
| `procedure_type` | `str` | 这一步对应哪类 `ProcedureRuntime` |
| `results_backend` | `str` | 结果当前落在哪个正式后端 |
| `results_path` | `Path | None` | 文件后端下结果文件的位置 |
| `export_format` | `str | None` | 是否额外导出了 `vtk` 等工件 |
| `export_path` | `Path | None` | 导出工件路径 |
| `frame_count` | `int` | 结果步里写出了多少帧 |
| `history_count` | `int` | 当前结果里累计写出多少历史量序列 |
| `summary_count` | `int` | 当前结果里累计写出多少摘要项 |
| `monitor_messages` | `tuple[str, ...]` | 作业执行过程中记录的消息流 |

最小消费方式通常就是读取这几个字段，而不是去猜文件命名：

```python
report = session.run_input_file(
    "cases/beam.inp",
    results_path="cases/beam.results.json",
)

if report.results_path is not None:
    results = session.open_results(report.results_path)
    print(results.list_steps())

for message in report.monitor_messages:
    print(message)
```

这段代码说明两件事：

- 脚本层继续打开结果，最稳妥的做法是从 `report.results_path` 接着走，而不是手写同名路径推断。
- 运行日志不需要额外猜测来源；它已经被收进 `report.monitor_messages`。

### 3. Job Snapshot

#### 3.1 `snapshot` 的角色

当前正式 `snapshot` 不是简单的“另存一个 INP”。它是 pyFEM 在 GUI 主线里冻结运行上下文的标准工件。

`JobSnapshot` 记录的核心信息包括：

| 字段 | 含义 |
| --- | --- |
| `snapshot_id` | 快照唯一标识 |
| `snapshot_kind` | 当前实现中至少有 `export`、`run`、`derived_case` 三类 |
| `model_name` | 模型名 |
| `snapshot_path` | 冻结后的 `INP` 路径 |
| `manifest_path` | sidecar manifest 路径 |
| `results_path` | 默认或显式绑定的结果路径 |
| `source_model_path` | 来源模型路径 |
| `derived_case_path` | 若是派生算例，则记录派生路径 |
| `created_at` | 创建时间 |

从这个结构本身可以确认，`snapshot` 是作业上下文描述符，而不是单一文件名。

#### 3.2 写出快照

`JobSnapshotService.write_snapshot()` 的正式行为是：

1. 用 `InpExporter` 把当前模型写成 `snapshot_path`。
2. 如果没有显式给 `results_path`，默认生成同名 `.results.json`。
3. 生成同名 `.snapshot.json` manifest。
4. 把 `snapshot` 元数据以 UTF-8、缩进 2 空格的 JSON 写入 manifest。

因此，一次标准快照写出后，至少会关联三类工件：

- `xxx.inp`
- `xxx.results.json`
- `xxx.snapshot.json`

帮助文档里如果只写“Write INP 会导出一个 inp 文件”，就会遗漏正式快照链路中最重要的 manifest 和默认结果路径关系。

##### 最小 manifest 示例

下面是一个精简后的 manifest 结构示意。字段名来自当前正式实现，具体路径和值会随本地环境变化：

```json
{
  "snapshot_id": "3a1c9f...",
  "snapshot_kind": "run",
  "model_name": "beam-case",
  "snapshot_path": "cases/.pyfem_snapshots/beam.run.3a1c9f.inp",
  "manifest_path": "cases/.pyfem_snapshots/beam.run.3a1c9f.snapshot.json",
  "results_path": "cases/beam.results.json",
  "source_model_path": "cases/beam.inp",
  "derived_case_path": null,
  "created_at": "2026-04-17T08:00:00+00:00"
}
```

这个结构有三个阅读重点：

- `snapshot_path` 是冻结后的输入文件，不等于当前 GUI 正在编辑的活模型对象。
- `results_path` 是这份 snapshot 绑定的默认结果路径，后续 `run_snapshot()` 会继续沿用它。
- `manifest_path` 不是可选附件；它是 Job Center、Diagnostics 和复跑路径回查 snapshot 上下文的重要依据。

#### 3.3 运行快照

`run_snapshot()` 的正式行为非常明确：

- 先检查 `snapshot_path` 是否存在，不存在就直接报 `快照文件不存在: ...`
- 再通过 `JobManager.run_input_file()` 运行这个冻结文件
- 运行结束后，把 `last_run_report` 回写到 manifest 中

当前 manifest 里的 `last_run_report` 至少包含：

- `step_name`
- `procedure_type`
- `results_backend`
- `results_path`
- `export_format`
- `export_path`
- `frame_count`
- `history_count`
- `summary_count`

这说明 manifest 不只是“创建时的静态元数据”，还是一份会被复跑动作回写的运行记录。

#### 3.4 GUI 标准运行为什么先写快照

`GuiShell.submit_job()` 当前不会直接拿活模型求解。它会先做两件事：

1. 调用 `build_run_snapshot_path()` 构造标准运行快照路径。
2. 用 `snapshot_kind="run"` 把当前模型冻结下来，再通过 `run_snapshot()` 运行。

默认运行快照路径规则是：

- 如果当前模型有来源文件，就写到来源文件同目录下的 `.pyfem_snapshots/` 中，文件名形如 `<stem>.run.<id>.inp`
- 如果当前模型没有来源文件，就写到当前工作目录下的 `.pyfem_snapshots/`

这个设计的直接效果是：

- GUI 运行拿到的是冻结时刻的模型，而不是“运行中继续被编辑的活对象”
- 最近一次运行工件总能回溯到一份明确的 `snapshot`
- Job Center、Results / Output、Open Snapshot Manifest 这些入口都能围绕同一份工件集工作

回归测试也明确覆盖了这一点：快照冻结后，即使活模型随后继续变化，`run_snapshot()` 消费的仍然是冻结文件，而不是后来变动过的内存对象。

#### 3.5 Derived Case

`Derived Case` 不是普通的另存为。当前 `GuiShell.save_current_model_as_derived_case()` 的正式行为是：

1. 把当前活模型写成一个 `snapshot_kind="derived_case"` 的快照。
2. 在 `snapshot` 中记录 `derived_case_path`。
3. 清掉 `last_run_snapshot`。
4. 通过 `replace_loaded_model(..., source_path=target_path, mark_dirty=False)` 把当前打开模型的来源路径切换到新派生文件。

因此，`Derived Case` 的正式含义是：

- 形成一份新的源算例文件。
- 把当前 GUI 会话的“源路径”切换过去。
- 让后续默认结果路径、默认 VTK 路径、后续 Run Snapshot 语义都围绕这份新派生算例展开。

帮助文档不应把它写成单纯的“复制当前模型文件”。

#### 3.6 显式写快照与最近快照

GUI 中还有一条显式写出路径：`write_current_model_snapshot()`。

它的行为是：

- 使用 `snapshot_kind="export"` 写出显式快照
- 更新 `last_export_snapshot`

而 `latest_snapshot()` 会在 `last_export_snapshot` 与 `last_run_snapshot` 中按 `created_at` 取最新值。于是：

- `Run Last Snapshot`
- `Open Snapshot Manifest`

这些入口操作的不是“最后一次运行结果”这么简单，而是“当前 GUI 状态里时间上最新的可复用快照”。

#### 3.7 外进程求解请求

`GuiShell.build_run_process_request()` 提供了标准外进程求解请求。它当前会调用：

- 当前 Python 解释器
- `run_case.py`

并通过环境变量传递正式参数。当前已确认的环境变量包括：

- `PYFEM_INP_PATH`
- `PYFEM_MODEL_NAME`
- `PYFEM_RESULTS_PATH`
- `PYFEM_WRITE_VTK`
- `PYFEM_REPORT_PATH`
- `PYTHONIOENCODING=utf-8`
- `PYTHONUTF8=1`
- 可选的 `PYFEM_STEP_NAME`
- 可选的 `PYFEM_VTK_PATH`

因此，GUI 的外进程运行并不是一条“第二套私有协议”，而是对正式运行参数的标准化封装。

##### 最小 snapshot / derived case 示例

如果你想把当前 GUI 中的活模型固定下来，最小调用关系如下：

```python
# 显式写一个可回看的 snapshot
shell = GuiShell()
snapshot = shell.write_current_model_snapshot("cases/beam-export.inp")

print(snapshot.snapshot_kind)   # export
print(snapshot.snapshot_path)   # cases/beam-export.inp
print(snapshot.manifest_path)   # cases/beam-export.snapshot.json
```

这条路径的重点是“冻结并留档”，而不是立即求解。它会更新 `last_export_snapshot`，方便后续 `Run Last Snapshot` 或 `Open Snapshot Manifest`。

```python
# 保存为 Derived Case，并把当前源路径切到新算例
shell = GuiShell()
derived = shell.save_current_model_as_derived_case("cases/beam-variant.inp")

print(derived.snapshot_kind)             # derived_case
print(shell.state.opened_model.source_path)
```

这条路径的重点是“派生出一个新的算例分支”。源码回归测试已经覆盖：派生之后，当前 GUI 会话的来源路径会切到这份新文件，后续默认结果路径也随之变化。

### 4. 作业监控与报告

#### 4.1 Monitor 机制

当前 `JobMonitor` 只有一个最小接口：

- `message(text: str) -> None`

正式内置实现有两个：

- `InMemoryJobMonitor`
- `ConsoleJobMonitor`

它们分别对应：

- 在内存中积累消息，最终进入 `JobExecutionReport.monitor_messages`
- 在终端直接输出 `[job] ...`

因此，作业监控在当前实现里是“消息流”，不是复杂的事件总线。

#### 4.2 `JobExecutionReport`

一次正式作业结束后，返回的是 `JobExecutionReport`。当前它至少会携带：

- `model_name`
- `job_name`
- `step_name`
- `procedure_type`
- `results_backend`
- `results_path`
- `export_format`
- `export_path`
- `frame_count`
- `history_count`
- `summary_count`
- `monitor_messages`

这意味着 `JobExecutionReport` 既是“运行完成通知”，也是 GUI 和脚本层继续定位结果工件的标准入口。

`JobExecutionReport` 在主线里通常这样被消费：

```python
report = shell.submit_job(results_path="cases/beam.results.json")

print(report.step_name)
print(report.results_path)
print(report.monitor_messages)
```

这段代码里最重要的不是 `print()` 本身，而是消费顺序：

1. 先从 `report` 读出这次运行到底执行了哪个步骤。
2. 再从 `report.results_path` 打开正式结果。
3. 如果需要看消息流或给 GUI 对话框补摘要，则继续消费 `report.monitor_messages`。

因此，`report` 的作用不是替代结果数据库，而是把“这次运行发生了什么、结果在哪里、是否还有额外导出”一次性交给上层。

#### 4.3 GUI 监控入口

GUI 中与正式作业链路直接相连的入口包括：

- `Job Center...`
- `Monitor Current Run...`
- `Diagnostics...`
- `Results / Output...`
- `Run Last Snapshot`
- `Open Snapshot Manifest`

其中：

- `Job Center` 负责展示记录和执行动作
- `Job Monitor` 负责显示当前或最近一次运行的消息
- `Job Diagnostics` 负责给出问题、建议、警告、工件状态摘要
- `Results / Output` 负责围绕结果、报告、manifest 打开对应工件

从对话框实现还能确认，`Job Center` 右侧动作区当前直接包含：

- `Run Last Snapshot`
- `Monitor`
- `Open Results`
- `Open Snapshot Manifest`
- `Open Report`
- `Save As Derived Case`
- `Re-run`

因此，帮助文档中可以把 Job Center 写成 GUI 作业总入口，而不是单纯的“历史列表窗口”。

#### 4.4 Diagnostics 的正式边界

需要特别收口的一点是：

- `Job Diagnostics` 已经是正式 GUI 入口
- 但 `Step Diagnostics...` 仍是 placeholder，代码里明确写着 “Formal step diagnostics are not supported yet.”

所以文档里只能写：

- 已正式支持 Job 级 diagnostics 摘要
- 尚未正式支持 Step 级 diagnostics

不能把两者混成“已经有完整诊断系统”。

### 5. 边界与常见误解

下表用于澄清本章最容易混淆的几组对象或入口。

| 对象或入口 | 它是什么 | 它不是什么 | 最容易混淆的点 |
| --- | --- | --- | --- |
| `JobExecutionRequest` | 一次标准作业的输入描述 | 运行后的结果对象 | 它只描述“要怎么跑”，不保存帧数和结果摘要 |
| `JobExecutionReport` | 一次运行结束后的标准返回对象 | 结果数据库本体 | 它告诉你结果在哪里，但不直接代替 `ResultsDatabase` |
| `snapshot` | 冻结后的输入与上下文描述符 | 活模型对象本身 | 快照生成后，后续运行消费的是冻结文件，不是继续变化的内存对象 |
| `manifest` | snapshot 的 sidecar 元数据文件 | `.results.json` 正式结果文件 | manifest 记录上下文与最近运行摘要，结果场数据仍在结果文件里 |
| `Run Current Model` | 先写 `run` snapshot，再基于 snapshot 运行 | 直接拿 GUI 活模型求解 | GUI 主线故意先冻结，避免“边编辑边求解”带来的不确定性 |
| `Derived Case` | 生成新算例并切换当前来源路径 | 普通的另存副本 | 它会改变后续默认结果路径和后续 snapshot 语义 |

本章里最常见的误解有四类：

1. 看到 `write_snapshot()` 就以为只是导出一个 `.inp` 文件。实际上它还会同步生成 `.snapshot.json`，并绑定默认 `results_path`。
2. 看到 `JobExecutionReport` 就以为结果已经全部塞进这个对象里。实际上真正的场数据、历史量和摘要仍然在结果层对象里。
3. 看到 GUI 的 `Run Current Model` 就以为它绕过 snapshot 直接求解。当前正式主线恰恰相反，它会先冻结成 `run` snapshot。
4. 把 `Derived Case` 当成“和 snapshot 一样的导出动作”。两者都写文件，但 `Derived Case` 还会切换 GUI 当前源路径，这是它和普通导出最大的区别。

### 6. 使用建议

- 需要最稳定、最可追溯的运行工件时，优先走 `snapshot -> run_snapshot -> manifest/results/report` 这条正式链路。
- GUI 中需要反复复跑时，优先使用 `Run Last Snapshot`，而不是依赖“当前活模型一定没变”这种假设。
- 需要形成新算例分支时，优先使用 `Derived Case`，因为它会同步切换当前源路径。
- 在脚本中直接运行 `ModelDB` 时，如果选择 `json` 后端或 `vtk` 导出，务必显式提供路径。

### 7. 本章主线图

```text
INP / ModelDB
    -> JobExecutionRequest
    -> JobManager.execute()
    -> ProcedureRuntime.run()
    -> JobExecutionReport

GUI 活模型
    -> JobSnapshotService.write_snapshot()
    -> snapshot + manifest
    -> run_snapshot()
    -> results / optional vtk / report
```

---

## 第 14 章 插件与扩展说明

本章面向维护者和扩展开发者。重点不是畅想未来插件生态，而是明确当前代码里已经形成正式合同的扩展边界。

### 1. 功能定位

当前 pyFEM 的扩展主线围绕 `RuntimeRegistry` 建立。它承担两类职责：

- 统一保存运行时 provider 与 IO 工厂。
- 统一保存插件 manifest 与自由度布局元数据。

这意味着：

- 新扩展想进入正式编译/求解/导入/导出链路，最终都要挂到同一个 registry 上。
- `plugin.json` 本身不是运行时；真正生效的是“manifest + register_entry + RuntimeRegistry 注册动作”的组合。

### 2. `RuntimeRegistry`

#### 2.1 内置注册内容

当前内置 provider 和工厂已经在 registry 初始化时完成注册。正式内置内容可以概括为：

| 类别 | 当前内置 key |
| --- | --- |
| 材料 | `linear_elastic`、`elastic_isotropic`、`j2_plastic`、`j2_plasticity` |
| 截面 | `solid`、`plane_stress`、`plane_strain`、`beam` |
| 单元 | `C3D8`、`CPS4`、`B21` |
| 约束 | `displacement` |
| 相互作用 | `noop`、`none` |
| 分析过程 | `static`、`static_linear`、`static_nonlinear`、`modal`、`implicit_dynamic`、`dynamic` |
| importer | `inp` |
| results writer | `json`、`memory` |
| results reader | `json` |
| exporter | `vtk` |

这张表很重要，因为它给出了“当前不依赖插件也能工作的正式最小能力集合”。

#### 2.2 自由度布局

当前默认自由度布局不需要通过插件显式注册；`DofLayoutRegistry` 初始化时已经带了三种正式布局：

- `C3D8 -> (UX, UY, UZ)`
- `CPS4 -> (UX, UY)`
- `B21 -> (UX, UY, RZ)`

而 `RuntimeRegistry.register_dof_layout()` 允许扩展单元类型继续声明自己的节点自由度布局。

因此，单元扩展不仅要注册 element provider，还应同步考虑自由度布局是否需要一起声明。

##### 最小 registry 注册骨架

如果你只想先理解“插件到底向哪个对象注册什么”，最小骨架可以先看成下面这样。这个例子只强调调用关系，具体 provider/factory 实现由插件自己提供：

```python
def register_plugin(registry: RuntimeRegistry) -> None:
    registry.register_material("my_material", material_provider)
    registry.register_element("MYE4", element_provider)
    registry.register_dof_layout("MYE4", ("UX", "UY"))
    registry.register_importer("my_inp", importer_factory)
```

这段骨架里每一行都对应当前正式注册面里的一个入口：

- `register_material()`、`register_element()` 这类入口把 provider 注册进编译与 runtime 主线。
- `register_dof_layout()` 补的是编译层自由度布局，不会自动从 element provider 里“猜出来”。
- `register_importer()`、`register_exporter()` 这类入口注册的是工厂，不是 runtime provider。

读这段骨架时，最重要的不是 `my_material` 或 `MYE4` 这些示例 key，而是看清“插件真正生效”的地方始终是 `RuntimeRegistry`。

#### 2.3 可扩展注册点

当前 `RuntimeRegistry` 对外暴露的正式注册入口包括：

- `register_material()`
- `register_section()`
- `register_element()`
- `register_constraint()`
- `register_interaction()`
- `register_procedure()`
- `register_importer()`
- `register_results_writer()`
- `register_results_reader()`
- `register_exporter()`
- `register_plugin_manifest()`
- `register_dof_layout()`

这也定义了当前插件可进入的正式边界：

- 运行时对象层：材料、截面、单元、约束、相互作用、分析过程
- IO 层：导入器、结果后端、结果读取器、导出器
- 元数据层：plugin manifest、自由度布局

帮助文档不应把未暴露注册入口的概念对象写成“已有正式插件扩展点”。

#### 2.4 注册语义

当前 registry 中有两种不同的注册语义：

- provider 注册：例如材料、截面、单元、过程，这些注册进去的是运行时提供者
- factory 注册：例如 importer、writer、reader、exporter，这些注册进去的是工厂或可调用构造器

同时，插件 manifest 注册与具体扩展注册不是一回事：

- 只有在扩展点已经通过 `RuntimeRegistry` 注册成功后，manifest 才会被挂入 `_plugin_manifests`

`manifest` 的作用是声明扩展自身信息，不代替真实注册动作。

### 3. `plugin.json` 清单结构

#### 3.1 顶层结构

当前正式插件清单版本是：

- `manifest_version = "1"`

顶层 `PluginManifest` 当前包含这些正式字段：

| 字段 | 作用 |
| --- | --- |
| `name` | 插件名，不能为空 |
| `version` | 插件版本号，不能为空 |
| `manifest_version` | 当前仅支持版本 `1` |
| `compatibility` | 核心版本兼容范围 |
| `extensions` | 扩展点声明数组 |
| `register_entry` | 正式注册入口，形如 `module:function` |

其中：

- `name` 缺失会直接报错
- `version` 缺失会直接报错
- `manifest_version` 不是 `1` 也会直接 fail-fast
- `register_entry` 如果存在，必须写成 `module:function`

#### 3.2 扩展点声明

单个扩展点由 `PluginExtensionManifest` 描述。当前正式字段包括：

| 字段 | 作用 |
| --- | --- |
| `extension_type` | 扩展点类型 |
| `key` | registry 中使用的 key |
| `entry_point` | 扩展实现的入口路径 |
| `register_function` | 可选的登记函数名 |
| `metadata` | 附加元数据对象 |

当前正式支持的 `extension_type` 只有以下 10 种：

- `constraint`
- `element`
- `exporter`
- `importer`
- `interaction`
- `material`
- `procedure`
- `results_reader`
- `results_writer`
- `section`

因此，文档里不能把 `job`、`step`、`viewer`、`postprocess pipeline` 之类概念型名词写成“已经支持的 manifest 扩展类型”。

##### 最小 `plugin.json` 示例

下面是一个最小清单骨架。字段名和层次与当前正式实现一致，模块路径与 key 只是示意：

```json
{
  "name": "my-plugin",
  "version": "0.1.0",
  "manifest_version": "1",
  "compatibility": {
    "min_core_version": "0.1.0"
  },
  "register_entry": "my_plugin.registry:register_plugin",
  "extensions": [
    {
      "extension_type": "importer",
      "key": "my_inp",
      "entry_point": "my_plugin.io:MyInpImporter"
    }
  ]
}
```

这份清单最值得先记住三点：

- `register_entry` 是插件真正进入正式主线的入口；没有它，只有 `extensions` 声明还不够。
- `extensions[].extension_type` 必须落在当前正式支持的 10 种类型内。
- `extensions[].key` 最终应该和注册函数实际放进 `RuntimeRegistry` 的 key 对得上，否则注册校验会失败。

#### 3.3 版本兼容

`compatibility` 当前由 `PluginCompatibility` 表示，正式字段只有：

- `min_core_version`
- 可选的 `max_core_version`

其语义是：

- 只有 `min_core_version` 时：插件要求核心版本 `>= min_core_version`
- 同时给了 `max_core_version` 时：插件只兼容闭区间 `[min_core_version, max_core_version]`

这套合同是会被显式校验的，不是装饰性字段。

#### 3.4 `register_entry` 与 `entry_point` 的区别

这里很容易混淆，必须分开写：

- `register_entry`：插件正式注册函数入口，负责把扩展真正挂到 `RuntimeRegistry`
- `entry_point`：单个扩展声明指向的实现入口字符串
- `register_function`：扩展点条目里可选的登记函数名，属于条目级元数据

在当前实现里，真正决定插件能否进入正式主线的是 `register_entry` 能否成功导入、调用，并完成 registry 注册。

因此：

- 只写 `extensions[].entry_point`，不写 `register_entry`，不会形成正式注册
- `extensions` 里的声明如果没有被注册函数真正注册，也会在校验阶段被拦下

##### 最小注册函数骨架

顶层 `register_entry` 指向的函数，当前正式签名可以写成下面这样：

```python
def register_plugin(registry: RuntimeRegistry) -> None:
    registry.register_importer("my_inp", importer_factory)
```

这个函数的合同比看上去更严格：

- 参数必须只有一个。
- 参数必须是普通位置参数，不能改成关键字专用参数。
- 参数不能带默认值。
- 参数类型标注必须是 `RuntimeRegistry`。

如果插件作者把这个函数写成“无参数工具函数”“两个参数的初始化函数”或者“没有类型标注的自由函数”，当前注册主线都会在 discovery 阶段直接 fail-fast。

### 4. 插件发现与注册

#### 4.1 清单发现

`PluginDiscoveryService` 当前固定使用：

- 清单文件名：`plugin.json`

发现规则也很简单：

- 如果传入的是文件路径，就直接当作 manifest 文件
- 如果传入的是目录，就递归搜索目录下全部 `plugin.json`
- 如果路径不存在，直接报错

因此，当前正式插件发现能力是“本地文件/目录递归发现”，而不是在线仓库、包管理器或 GUI 市场。

#### 4.2 manifest 加载

manifest 加载阶段会做这些事情：

1. 按 UTF-8 读取文件
2. 解析 JSON
3. 要求根对象必须是 JSON object
4. 调用 `parse_plugin_manifest()` 转成正式 manifest 对象

所以即使文件名正确，下面这些情况也都会被拒绝：

- 不是合法 JSON
- 顶层不是对象
- 顶层字段缺失或字段类型不符合 manifest 合同

#### 4.3 注册流程

`register_into_registry()` 当前的正式顺序是：

1. 检查同名 plugin manifest 是否已经在 registry 中存在
2. 校验插件与 `registry.core_version` 的兼容性
3. 解析并加载 `register_entry`
4. 校验注册函数签名
5. 执行注册函数
6. 校验 manifest 声明的扩展点确实已经通过 `RuntimeRegistry` 注册
7. 调用 `registry.register_plugin_manifest(manifest)`

这个流程说明了两点：

- manifest 注册是在“真实扩展已经注册成功”之后才发生的
- registry 会主动检查“声明”和“实际注册结果”是否一致

#### 4.4 注册函数签名合同

当前注册函数签名要求非常明确：

- 必须只接收一个参数
- 该参数必须是普通位置参数
- 参数不能带默认值
- 参数类型标注必须是 `RuntimeRegistry`

这套签名要求是正式合同，不是代码风格建议。

#### 4.5 `PyFEMSession` 便捷入口

对于上层调用者，当前已有两条便捷 API：

- `discover_plugin_manifests(*search_roots)`
- `register_discovered_plugins(*search_roots)`

它们的作用分别是：

- 先只发现 manifest，供上层查看或决定是否继续注册
- 发现后立即把它们注册进当前 session 的 `registry`

因此，插件扩展已经不需要上层手动逐个拼 discovery 和 register 逻辑。

#### 4.6 最小发现与注册示例

如果你已经把 `plugin.json` 放到某个本地目录，最小发现与注册流程可以写成：

```python
session = PyFEMSession()

discovered = session.discover_plugin_manifests("plugins")
for item in discovered:
    print(item.path)
    print(item.manifest.name)

registered = session.register_discovered_plugins("plugins")
for manifest in registered:
    print(manifest.name)
```

这条链路里返回的是两种不同对象：

- `discover_plugin_manifests()` 返回 `DiscoveredPluginManifest`，重点是“从哪里发现了哪份 manifest”。
- `register_discovered_plugins()` 返回已经成功注册进当前 session `registry` 的 `PluginManifest`。

因此，如果你只是想先扫目录、列出候选插件，不必一上来就注册；如果你已经确认要启用插件，再走第二步注册即可。

#### 4.7 注册失败时先看哪一步

插件注册失败时，最省时间的办法不是先猜 provider 实现对不对，而是先看报错属于下面哪一层：

| 失败阶段 | 常见现象 | 应先检查什么 |
| --- | --- | --- |
| 发现阶段 | 路径不存在，或目录里根本没找到 manifest | 搜索根路径是否正确，文件名是否真是 `plugin.json` |
| 读取阶段 | manifest 不是合法 JSON，或根对象不是 JSON object | 文件编码、JSON 语法、顶层结构 |
| manifest 解析阶段 | 缺字段、`manifest_version` 不受支持、`extension_type` 不受支持 | `name`、`version`、`compatibility`、`extensions` 的格式 |
| 注册函数校验阶段 | 缺少 `register_entry`，或签名/类型标注不符合合同 | `register_entry` 字符串、函数参数个数、`RuntimeRegistry` 标注 |
| 实际注册阶段 | 注册函数运行后，registry 里找不到 manifest 声明的 key | 注册函数是否真的调用了对应的 `register_*()` 方法 |
| 兼容性阶段 | 当前核心版本不在插件声明范围内 | `min_core_version` / `max_core_version` 是否匹配当前 core 版本 |

当前源码和回归测试已经明确覆盖的 fail-fast 情况包括：

- manifest 路径不存在
- manifest 不是合法 JSON
- `register_entry` 缺失
- 注册函数签名不合法
- 注册函数参数没有标注 `RuntimeRegistry`
- manifest 声明的扩展点没有真正注册进 `RuntimeRegistry`
- 插件声明的核心版本范围与当前版本不兼容

### 5. 边界与常见误解

下表用于澄清本章最容易被写混的几组对象和概念。

| 对象或概念 | 它是什么 | 它不是什么 | 常见混淆点 |
| --- | --- | --- | --- |
| `RuntimeRegistry` | 正式统一注册表 | 插件本体或插件市场 | 插件是否生效，最终要看 registry 里有没有注册到位 |
| `plugin.json` | 插件声明与校验清单 | 真正执行注册动作的代码 | 清单只描述“我打算提供什么”，不会自动完成注册 |
| `register_entry` | 顶层正式注册函数入口 | 单个扩展实现类本身 | 它指向的是“把扩展挂进 registry 的函数”，不是 provider 类名 |
| `entry_point` | 单个扩展声明里的实现入口字符串 | 顶层注册入口 | 两者都长得像模块路径，但作用不同 |
| `extensions` | 插件自报的扩展清单 | registry 当前已经注册成功的事实列表 | 注册流程会检查“声明”和“实际注册结果”是否一致 |
| `register_dof_layout()` | 单元自由度布局扩展入口 | element provider 的别名 | 自由度布局需要单独声明，不会自动从 provider 推断 |

本章里最常见的误解有五类：

1. 以为写完 `plugin.json` 就等于插件已经进入正式主线。实际上没有成功执行 `register_entry`，扩展并不会真正生效。
2. 以为 `extensions[].entry_point` 会替代顶层 `register_entry`。当前实现里，真正触发注册的是顶层 `register_entry`。
3. 以为插件可以直接扩展任意 GUI 或 job 行为。当前正式边界仍然是“通过 `RuntimeRegistry` 扩展 runtime 与 IO”。
4. 以为 element 扩展只要注册 provider 就够了。若节点自由度布局不同，还必须同步处理 `register_dof_layout()`。
5. 以为 manifest 里的扩展声明只是说明文字。当前注册流程会显式校验这些声明是否真的注册成功。

### 6. 当前边界与建议

- 当前插件扩展边界是“进入 RuntimeRegistry”，不是“自由注入任意 GUI/脚本行为”。
- `plugin.json` 当前主要承担声明和校验责任；真正改变系统行为的是注册函数。
- 编写插件时，应先确认 manifest 里的每个扩展声明最终都能在 registry 中被查到，否则会在正式注册阶段被拦截。
- 版本兼容字段应当认真维护，因为当前实现会严格校验核心版本范围。

### 7. 本章主线图

```text
plugin.json
    -> PluginDiscoveryService.discover()
    -> parse_plugin_manifest()
    -> register_entry(registry)
    -> RuntimeRegistry.register_*(...)
    -> register_into_registry() consistency check
    -> registry.register_plugin_manifest(manifest)
```

---

## 第 15 章 常见错误与排查

本章只收录代码中已经存在明确异常消息、能力边界消息或 fail-fast 路径的场景。不做“可能的猜测性故障库”。

### 1. 输入与建模错误

#### 1.1 INP 翻译错误

当前 `InpImporter` 已经把不少“不在正式支持子集内”的输入拦在翻译阶段。最常见的几类如下：

| 典型消息 | 常见原因 | 建议处理 |
| --- | --- | --- |
| `当前不支持顶层关键字 ...` | 出现了 importer 尚未收口的顶层关键字 | 删去该关键字，或改写成当前正式支持的结构化对象 |
| `部件 ... 缺少 END PART。` | `*Part` 块没有正确结束 | 检查 `*End Part` 是否缺失或拼写错误 |
| `装配 ... 缺少 END ASSEMBLY。` | `*Assembly` 块没有正确结束 | 检查 `*End Assembly` |
| `当前仅支持空的 SYSTEM 关键字；带坐标变换定义的 SYSTEM 尚未纳入正式 importer 主线。` | 使用了带数据的 `SYSTEM` 块 | 改为当前 importer 可接受的空控制关键字，或改走别的建模表达 |
| `当前仅支持不带数据行的 RESTART 控制关键字。` | `RESTART` 用法超出当前子集 | 收缩到无数据行控制形式 |
| `当前仅支持 variable=PRESELECT 且不带数据行的 Abaqus OUTPUT 控制关键字。` | 使用了更复杂的 Abaqus 输出控制语法 | 改用 pyFEM 自己的 `OutputRequest` 主线 |
| `当前输入同时包含多个部件，旧式关键字缺少明确 scope，无法安全推断所属部件。` | 旧式关键字在多部件模型里缺少作用域 | 显式补足 scope，或改成结构化定义 |
| `当前一个模型仅支持一个 ASSEMBLY 块。` | 输入里定义了多个装配块 | 收敛为一个装配块 |

关于 `BOUNDARY/CLOAD/DSLOAD`，当前 importer 也有明确的结构化约束：

- `结构化 BOUNDARY 定义必须按 DOF, VALUE 书写。`
- `结构化 CLOAD 定义必须按 COMPONENT, VALUE 书写。`
- `结构化 DSLOAD 定义必须按 COMPONENT, VALUE 书写。`
- `DSLOAD 定义至少需要给出表面名、载荷类型和值。`
- `目标表面 ... 不存在。`
- `当前 *Dsload 仅支持 P 压力载荷，收到 ...`

如果装配模型中的目标写法不规范，还会看到：

- `装配模型中的目标 ... 必须显式写成 INSTANCE.NAME，或先在 ASSEMBLY 中定义唯一别名。`
- `当前不支持 INP 自由度编号 ...`
- `当前不支持 INP 载荷自由度编号 ...`

这些错误的共同特点是：它们通常出现在“还没开始求解”之前，属于输入翻译失败，而不是数值求解失败。

#### 1.2 `ModelDB` 校验错误

即使不走 importer，直接构造 `ModelDB` 也会经过正式模型校验。当前常见 fail-fast 场景包括：

| 典型消息 | 含义 | 建议处理 |
| --- | --- | --- |
| `模型 ... 至少需要一个部件定义。` | 模型骨架不完整 | 至少先建立一个 `Part` |
| `部件 ... 中单元 ... 引用了不存在的截面 ...` | 单元与截面引用断裂 | 检查 `section_name` |
| `部件 ... 中单元 ... 引用了不存在的材料 ...` | 单元直绑材料但材料未定义 | 检查 `material_name` |
| `截面 ... 引用了不存在的作用域 ...` | 截面 scope 错误 | 检查 `scope_name` |
| `截面 ... 引用了不存在的材料 ...` | 截面材料引用断裂 | 检查 `material_name` |
| `截面 ... 引用了不存在的区域 ...` | 区域名或作用域不匹配 | 检查 `region_name + scope_name` |
| `输出请求 ... 缺少目标名称。` | 非 `model` 级请求但没给 `target_name` | 补足目标名称 |
| `输出请求 ... 引用了不存在的作用域 ...` | `scope_name` 不可解析 | 检查作用域是否存在 |
| `分析步骤 ... 必须给出 procedure_type。` | 步骤类型未声明 | 补足 `procedure_type` |
| `分析步骤 ... 引用了不存在的边界条件/节点载荷/分布载荷/输出请求 ...` | 步骤引用链断裂 | 检查步骤引用列表 |
| `作业 ... 至少需要引用一个分析步骤。` | `JobDef` 为空壳 | 至少绑定一个步骤 |
| `作业 ... 引用了不存在的分析步骤 ...` | 作业与步骤表不一致 | 检查 `job.step_names` |

此外，几何和网格对象自身也会做严格校验，例如：

- 同一网格中的节点空间维度必须一致
- 单元不能引用不存在的节点
- 表面不能引用不存在的单元
- 方向 `axis_1` 与 `axis_2` 必须线性独立
- 刚体变换旋转矩阵必须是正交右手系矩阵

所以，如果模型是通过 Python API 组装出来的，排查顺序应优先放在“对象关系是否完整”，而不是先怀疑求解器。

### 2. 求解错误

#### 2.1 运行请求与步骤选择错误

还没进入数值求解之前，作业层自己就会拦住一批配置错误：

| 典型消息 | 含义 | 建议处理 |
| --- | --- | --- |
| `JobExecutionRequest 必须且只能提供 input_path 或 model 之一。` | 来源不唯一或为空 | 二选一，别同时给 |
| `仅在指定 export_format 时才允许提供 export_path。` | 导出参数不成套 | 同时给全导出参数 |
| `JSON results backend 需要 results_path，或提供 input_path 以生成默认路径。` | 直跑 `ModelDB` 时少了结果路径 | 给 `results_path`，或改用 `memory` |
| `导出请求需要 export_path，或提供 input_path 以生成默认导出路径。` | 导出路径无法推导 | 显式给 `export_path` |
| `模型中不存在步骤 ...` | 指定了不存在的步骤 | 检查 `step_name` |
| `当前作业包含多个步骤，请显式指定 step_name。` | 一个 job 里有多个候选步骤 | 显式指定 |
| `当前模型中没有可运行的步骤。` | 没有步骤 | 先定义步骤 |
| `当前模型包含多个步骤，请显式指定 step_name。` | 模型里有多个步骤但没有指定 | 显式指定 |

这些错误出现时，优先检查作业请求本身，而不是继续追数值参数。

#### 2.2 非线性主线错误

当前 `static_nonlinear + nlgeom=True` 有非常明确的正式收口。最重要的边界可以概括为：

- 正式支持的 `nlgeom` 单元主线仅包括 `B21 corotational`、`CPS4 total_lagrangian`、`C3D8 total_lagrangian`
- 正式支持的 `nlgeom` 材料子集仅包括 `B21 + 线弹性/J2`、`CPS4 + plane_strain + 线弹性/J2`、`C3D8 + solid + 线弹性/J2`
- 正式支持的 `nlgeom` 载荷范围仅包括位移边界与 `nodal load`

因此，下面这些报错都属于“主线边界明确拒绝”，不是隐藏 bug：

- `静力非线性步骤 ... 请求 nlgeom=True，但存在未纳入正式主线的单元类型: ...`
- `静力非线性步骤 ... 请求 nlgeom=True，但存在未支持的单元/截面/材料组合: ...`
- `静力非线性步骤 ... 请求 nlgeom=True，但以下 distributed load 当前未纳入正式主线: ...`
- `当前暂不支持 nlgeom=True 下的 distributed load、surface pressure 或 follower pressure 当前构形语义。`

这类问题的正确修复方向通常不是“调容差”，而是：

- 先把模型退回正式支持的单元/材料/载荷组合
- 或者把分析改成线性小变形主线

#### 2.3 非线性参数错误

当前静力非线性参数和隐式动力学参数都做了显式校验。常见报错包括：

- `隐式动力学时间步长必须大于零。`
- `隐式动力学步数必须为正整数。`
- `静力非线性参数 max_increments 必须为正整数。`
- `静力非线性参数 initial_increment 必须位于 (0, 1]。`
- `静力非线性参数 min_increment 必须位于 (0, initial_increment]。`
- `静力非线性参数 max_iterations 必须为正整数。`
- `静力非线性参数 ... 必须为布尔值。`

因此，遇到静力非线性或隐式动力学无法启动时，应先检查参数字典的类型和值域，而不是先判断“求解器收敛不好”。

#### 2.4 材料与单元组合错误

当前材料与单元组合也有明确的 fail-fast 边界：

- `单元 ... 当前仅正式支持 C3D8 + nlgeom + 线弹性，以及 C3D8 + nlgeom + J2；暂不支持材料类型 ...`
- `单元 ... 当前仅正式支持 CPS4 + nlgeom + plane_strain + J2，暂不支持 CPS4 + nlgeom + plane_stress + J2。`
- `单元 ... 当前仅正式支持 CPS4 + nlgeom + 线弹性，以及 CPS4 + nlgeom + plane_strain + J2；暂不支持材料类型 ...`
- `J2 塑性材料的 young_modulus 必须大于零。`
- `J2 塑性材料的 poisson_ratio 必须位于 (-1, 0.5)。`
- `J2 塑性材料的 yield_stress 必须大于零。`

这里最值得强调的是：

- `CPS4 + plane_stress + J2 + nlgeom` 不是“暂时测试少”，而是代码里明确写出来的不支持组合
- J2 参数错误是材料运行时的前置校验，不属于计算到中途才发现的问题

#### 2.5 载荷与单元边界错误

当前分布载荷与表面载荷也有明确边界：

- `单元类型 ... 尚未实现表面分布载荷。`
- `C3D8 当前仅支持 pressure 类型表面载荷，收到 ...`
- `当前载荷装配暂不支持目标类型 ...`
- `当前不支持载荷分量 ...`

这些错误通常说明当前模型试图让载荷走进一条尚未纳入正式主线的单元/载荷组合。排查时应优先确认：

- 目标类型是不是当前支持的 `node/node_set` 或正式表面载荷对象
- 载荷分量是否属于当前单元和载荷类型允许的集合
- 是否错误地把非正式 `distributed load` 组合带进了 `nlgeom=True` 场景

### 3. 结果与导出错误

#### 3.1 输出请求错误

输出请求在编译阶段就会做强约束。当前最常见的几类报错如下：

| 典型消息 | 含义 | 建议处理 |
| --- | --- | --- |
| `输出请求 ... 的 frequency 必须大于零。` | 频率非法 | 改为正整数 |
| `输出请求 ... 至少需要声明一个变量。` | 变量列表为空 | 至少声明一个字段 |
| `步骤 ... 的过程类型 ... 暂不支持输出变量 ...` | 变量与过程类型不匹配 | 改为该步骤支持的变量 |
| `输出请求 ... 中变量 ... 只能写在位置 ...，不能写在 ...。` | 变量位置写错 | 把变量放回正式位置 |
| `历史量输出不允许声明 target_name。` | 历史量输出不接受目标名 | 删去 `target_name` |
| `历史量输出当前不允许声明 scope_name。` | 历史量输出不接受作用域 | 删去 `scope_name` |
| `节点输出请求缺少 target_name。` / `节点集合输出请求缺少 target_name。` | 节点类输出没给目标 | 补足目标 |
| `单元输出请求缺少 target_name。` / `单元集合输出请求缺少 target_name。` | 单元类输出没给目标 | 补足目标 |
| `输出请求未解析到任何节点目标: ...` / `输出请求未解析到任何单元目标: ...` | 目标解析为空 | 检查 target 和 scope 是否正确 |
| `当前尚未实现输出位置 ...` | 使用了未进入正式主线的位置 | 改用当前支持的位置 |

因此，输出请求问题大多属于“编译合同错误”，而不是“结果写出器坏了”。

#### 3.2 结果浏览与 probe 错误

结果数据库、结果 facade 和 probe 侧当前也有明确错误消息。常见场景包括：

- `结果数据库中不存在步骤 ...`
- `步骤 ... 中不存在帧 ...`
- `步骤 ... 中不存在历史量 ...`
- `步骤 ... 中不存在摘要 ...`
- `步骤 ... 中不存在请求的 frame_ids=(...)。`
- `步骤 ... 的帧 ... 中不存在结果场 ...`
- `结果场 ... 中不存在目标 ...`
- `结果场 ... 中不存在分量 ...`
- `结果场 ... 的目标 ... 缺少分量 ...`
- `结果场 ... 的目标 ... 为标量结果，不支持分量 ...`

这类错误的排查顺序建议是：

1. 先确认 `step_name` 对不对。
2. 再确认 `frame_id` 是否真的存在。
3. 再确认该帧里有没有这个 `field_name`。
4. 最后再看 `target_key`、`component_name` 是否匹配。

probe 报错通常表示访问路径不正确，不一定代表求解结果本身有问题。

#### 3.3 导出、快照与 GUI 结果错误

当前结果导出和 GUI 结果入口还有几类常见边界错误：

| 典型消息 | 含义 | 建议处理 |
| --- | --- | --- |
| `分析步骤 ... 的 procedure_type=... 当前无法导出。` | InpExporter 不支持该步骤类型导出 | 改用当前正式可导出的步骤类型 |
| `当前 VTK 导出暂不支持单元类型 ...` | exporter 不认识该 cell type | 改用当前正式单元，或先扩展 exporter |
| `当前 GUI 视图区暂不支持单元类型 ...` | GUI 视图区几何构建不支持该单元 | 先走结果文件消费或扩展 GUI 视图区 |
| `当前 GUI 壳没有可用的结果路径。` | 还没有当前结果上下文 | 先运行一次 job 或显式打开结果 |
| `快照文件不存在: ...` | snapshot 工件已丢失 | 检查 `.pyfem_snapshots` 或显式快照文件是否还在 |
| `snapshot manifest 不存在: ...` | manifest sidecar 丢失 | 检查 `.snapshot.json` |
| `当前 GUI 壳没有可复用的 snapshot。` | 还没有形成最近快照 | 先 Write INP、Run Job 或 Save As Derived Case |
| `外部求解进程未生成运行报告。` | 外进程跑完后没有留下 report | 优先检查外进程错误、路径权限与报告路径 |

这些错误的共同点是：它们通常发生在“结果工件管理”和“GUI 工件消费”阶段，而不是求解器核心算法阶段。

### 4. 排查顺序建议

当你面对一个 pyFEM 运行失败时，当前最稳妥的排查顺序是：

1. 先看是不是输入/建模错误：关键字、scope、target、引用链是否完整。
2. 再看是不是作业请求错误：输入来源、步骤选择、结果路径、导出路径是否齐全。
3. 再看是不是正式主线边界：`nlgeom`、材料/截面/单元组合、载荷类型是否越界。
4. 最后才看结果浏览与导出：输出请求位置、probe 访问路径、VTK 单元映射、快照工件是否齐全。

这种顺序和当前代码结构是一致的，因为系统本来就是按“导入/校验 -> 编译 -> 求解 -> 结果消费”的层次逐层 fail-fast。

## 第 16 章 开发与维护说明
本章属于开发视角。它不重复前面章节的对象说明，而是回答四个维护问题：

1. 源码模块边界与帮助文档章节边界如何对齐。
2. 哪些对象属于正式主线，哪些只属于内部实现。
3. provider、runtime、solver、results、壳层应如何分仓说明。
4. 哪些写法必须避免，避免把测试对象、占位对象或未来规划写成正式支持。

### 1. 模块边界与章节边界

下表用于对齐“官方模块边界”和“帮助文档章节边界”。

| 模块 / 家族 | 正式职责 | 文档主章 | 维护口径 |
| --- | --- | --- | --- |
| 定义对象家族 | 保存模型、网格、装配、步骤、作业定义 | 第 4、8 章 | 只讲定义，不讲求解内部对象 |
| 编译桥接家族 | 解析作用域、自由度、provider 绑定并产出 `CompiledModel` | 第 4 章 | 只讲编译，不代讲 kernel / procedures / solver |
| kernel runtime 家族 | 定义局部物理对象与最小接口 | 第 10 章 | 只讲局部物理，不讲步骤推进或全局离散系统 |
| procedures 家族 | 组织单步执行、结果写出和状态继承 | 第 9 章 | 只讲步骤编排，不代替 solver |
| solver 家族 | 形成离散系统并管理 trial / committed state | 第 11 章 | 只讲全局离散求解，不代替产品入口 |
| 结果对象家族 | 定义结果协议、写读接口与消费链 | 第 12 章 | 只讲结果本体与消费接口，不回头代讲求解 |
| 产品壳层家族 | 向用户提供运行、打开结果、导出、GUI 入口 | 第 6、7、13 章 | 只讲入口与消费，不暴露 solver 内部合同 |

这一表的维护意义是：后续任何新增内容，都应先决定它属于哪个家族，再决定放到哪一章，而不是继续向第 4 章或 API 章堆。

### 2. 正式主线与内部对象

当前正式主线可压缩为：

```text
ModelDB
    -> Compiler
    -> CompiledModel
    -> kernel runtimes
    -> ProcedureRuntime
    -> Assembler / DiscreteProblem / RuntimeState
    -> ResultsWriter
    -> ResultsDatabase
    -> ResultsFacade / Query / Probe / GUI
```

只要某个对象或能力没有进入这条闭环，就不应轻率写成“正式主线”。

下表用于区分正式主线对象与内部对象。

| 对象 | 所属层 | 当前定位 | 是否适合普通用户直接依赖 | 维护说明 |
| --- | --- | --- | --- | --- |
| `ModelDB` | 定义层 | 正式定义对象 | 否，通常通过输入文件或高阶脚本间接使用 | 可作为建模 API 说明对象 |
| `CompiledModel` | 编译层 | 正式编译产物 | 否 | 适合作为高级脚本和维护对象说明 |
| `MaterialRuntime` 等 kernel 对象 | kernel 层 | 正式内部 runtime | 否 | 适合作为开发与架构说明对象 |
| `ProcedureRuntime` | procedures 层 | 正式内部过程对象 | 否 | 可写职责和方法，不应宣传为普通用户入口 |
| `Assembler`、`DiscreteProblem`、`RuntimeState` | solver 层 | 正式内部求解对象 | 否 | 面向开发者，不是产品壳层接口 |
| `ResultsDatabase` | 结果层 | 正式结果本体 | 一般否 | 高级脚本可直接读取 |
| `ResultsFacade` | 结果层消费侧 | 正式结果入口 | 是 | 普通用户和 GUI 应优先停留在这一层 |
| `PyFEMSession` | 产品壳层 | 正式用户入口 | 是 | 当前最稳妥的 Python 主入口 |

### 3. provider、runtime 与 placeholder 的边界

下表用于澄清最容易被误写的扩展边界。

| 对象 | 它是什么 | 它不是什么 | 维护写法 |
| --- | --- | --- | --- |
| provider | 把定义对象构造成正式 runtime 的扩展入口 | 不是 runtime 本体 | 可以写成正式扩展点 |
| formal runtime | 已进入正式主线的运行时对象 | 不是未来规划占位对象 | 可以写成当前支持链路 |
| placeholder runtime | provider 缺失时的受控占位对象 | 不是正式支持能力 | 只能写成“占位”或“待接入” |

特别要避免把 placeholder 写成已支持，包括但不限于：

- `MaterialRuntimePlaceholder`
- `SectionRuntimePlaceholder`
- `ElementRuntimePlaceholder`
- `ConstraintRuntimePlaceholder`
- `InteractionRuntimePlaceholder`

placeholder 的意义是保留对象图和错误边界，不是替代正式实现。

### 4. 扩展点分层表

下表只说明“哪些地方可以扩展”，以及这些扩展分别落在哪一层。

| 扩展类别 | 正式入口 | 所属层 | 维护要求 |
| --- | --- | --- | --- |
| 输入扩展 | `register_importer()` | 产品壳层 / IO 边界 | 需要能产出 `ModelDB` |
| 编译桥接扩展 | `register_dof_layout()` | 编译层 | 需要能进入 `DofManager` 正式主线 |
| kernel 扩展 | `register_material()`、`register_section()`、`register_element()`、`register_constraint()`、`register_interaction()` | kernel / 编译边界 | 需要补齐 provider、runtime 和最小职责 |
| procedures 扩展 | `register_procedure()` | procedures / 编译边界 | 需要补齐 provider、过程对象、结果写出和状态边界 |
| 结果后端扩展 | `register_results_writer()`、`register_results_reader()` | 结果层 | 需要补齐协议读写、一致性和消费链 |
| 导出扩展 | `register_exporter()` | 产品壳层 / 消费边界 | 需要能消费结果层对象，而不是重造结果协议 |

如果一个新能力只完成了注册，但没有贯通到完整主线，也不应写成“当前正式支持”。

### 5. 方法边界与文档边界

开发文档除了区分对象层次，还必须区分“公开方法”和“内部 helper”。

| 类 / 家族 | 可以写成正式边界的方法 | 不应写成公共接口的方法类别 | 维护口径 |
| --- | --- | --- | --- |
| `PyFEMSession` | `load_model_from_file()`、`run_input_file()`、`run_model()`、`open_results()`、`export_results()` | 会话内部依赖注入与 `JobManager` 细节 | 壳层只暴露稳定入口 |
| `Compiler` | `compile()` | `_build_*` 这一类内部装配 helper | 编译层只承诺总入口 |
| `CompiledModel` | `get_element_runtime()`、`get_step_runtime()`、`iter_step_names()`、`publish_step_state()`、`resolve_inherited_step_state()` | 直接篡改 runtime 映射或快照容器 | 通过正式访问方法读写 |
| `ResultsFacade` | `session()`、`list_steps()`、`step()`、`frame()`、`field()`、`history()`、`summary()`、`query()`、`probe()` | `_iter_*`、`_filter_*` 这类 helper | 结果门面只公开 reader-only 入口 |
| `ResultsQueryService` | `steps()`、`frames()`、`field_overview()`、`histories()`、`summaries()` | 内部筛选 helper | 只写筛选和概览能力 |
| `ResultsProbeService` | `history()`、`field_component()`、`node_component()`、`export_csv()` | 帧选择和分量抽取 helper | 只写 probe 与导出能力 |

总原则只有一条：公开方法写“外部可依赖的语义”，私有 helper 留在实现层。

### 6. 当前禁止写法

下表用于约束后续文档维护。

| 禁止写法 | 原因 | 正确口径 |
| --- | --- | --- |
| 把 `kernel / procedures / solver` 再写成一个“运行时层大桶” | 会再次打散边界 | 按三层分别说明 |
| 把 `StepDef` 写成执行对象 | 混淆定义层与 procedures 层 | 写成“步骤定义对象” |
| 把 `ResultsFacade` 写成 `ResultsDatabase` | 混淆结果本体与结果门面 | 明确区分“结果本体”和“消费入口” |
| 把 `RuntimeState` 写成结果对象 | 混淆 solver 与结果层 | 写成“内部求解状态” |
| 把 placeholder runtime 写成正式支持 | 占位不等于正式实现 | 写成“占位对象”或“待接入” |
| 把测试夹具对象写成稳定 API | 证据不足 | 写成“测试对象”或标注“待确认” |
| 把 GUI 预留入口写成已支持功能 | 没有正式主线闭环 | 写成“预留入口”或“未来规划” |
| 把私有 `_build_*`、`_filter_*`、`_select_*` helper 写成公共方法 | 会把实现细节误写成合同 | 只写公开入口 |

### 7. 维护检查表

#### 7.1 文档检查

| 检查项 | 维护要求 |
| --- | --- |
| 是否先给层，再讲对象 | 先确定对象属于哪个家族和哪一层 |
| 是否把结果层和消费层分开 | `ResultsDatabase` 与 `ResultsFacade` 不得混用 |
| 是否把内部对象和壳层入口分开 | `Assembler`、`RuntimeState` 不得写成普通用户入口 |
| 是否把 placeholder 和未来规划分开 | 无源码闭环就不能写成当前支持 |
| 是否在证据不足处写明“待确认” | 保持保守口径 |

#### 7.2 实现检查

| 变更类型 | 最低建议 |
| --- | --- |
| 新 kernel 对象 | 补齐 provider、runtime、最小接口和边界说明 |
| 新 procedure | 补齐 provider、求解主线、状态继承和结果写出 |
| 新 solver 能力 | 先确认它属于内部对象还是正式外部边界 |
| 新结果后端 | 补齐 writer / reader / facade 消费链 |
| 新 GUI 结果消费 | 优先复用 `ResultsFacade -> Query / Probe -> GuiResultsViewContext` 链 |

### 8. 本章主线图

```text
定义对象家族
    -> 编译桥接家族
    -> kernel runtime 家族
    -> procedures 家族
    -> solver 家族
    -> 结果对象家族
    -> 产品壳层 / 消费层
```

---

## 第 17 章 术语表
本章是全文统一口径层。它既服务于用户阅读，也服务于维护者回查对象边界，因此每个术语都尽量回答四个问题：

1. 它属于哪一层。
2. 它主要面向用户还是开发者。
3. 推荐怎样解释。
4. 最容易和谁混淆。

### 1. 分层术语总表

| 术语 | 所属层 | 面向对象 | 推荐解释 | 容易和谁混淆 |
| --- | --- | --- | --- | --- |
| `ModelDB` | 定义层 | 用户 + 开发 | 模型定义数据库，保存部件、装配、步骤与作业定义 | `CompiledModel`、`ResultsDatabase` |
| `CompilationScope` | 编译层 | 开发为主 | 名称解析与 canonical scope 展开边界 | `scope_name`、`instance_name` |
| `CompiledModel` | 编译层 | 开发为主 | 已编译对象图与状态快照持有者 | `ModelDB`、`DiscreteProblem` |
| `kernel` | kernel 层 | 开发为主 | 材料、截面、单元、约束、相互作用的局部运行时层 | procedures、solver |
| `procedures` | procedures 层 | 开发为主 | 负责步骤编排、状态继承和结果写出的过程层 | kernel、solver |
| `solver` | solver 层 | 开发为主 | 负责全局离散系统、约束缩并和状态管理的求解层 | procedures、results |
| `results` | 结果层 | 用户 + 开发 | 负责结果协议、写读接口和结果消费链 | solver、GUI 缓存 |
| 产品壳层 / 消费层 | 壳层 | 用户 + 开发 | 负责统一运行入口、结果入口和 GUI 入口 | solver 内部对象 |

### 2. 对象家族术语表

#### 2.1 定义与编译家族

| 术语 | 所属层 | 面向对象 | 推荐解释 | 容易和谁混淆 |
| --- | --- | --- | --- | --- |
| 网格记录家族 | 定义层 | 用户 + 开发 | `NodeRecord`、`ElementRecord`、`Surface`、`Orientation` 等建模记录对象 | kernel runtime |
| 定义对象家族 | 定义层 | 用户 + 开发 | `Part`、`Assembly`、`StepDef`、`JobDef` 等正式定义对象 | 编译产物 |
| 编译桥接对象家族 | 编译层 | 开发为主 | `Compiler`、`DofManager`、`CompiledModel` 等桥接对象 | 产品壳层入口 |
| `StepDef` | 定义层 | 用户 + 开发 | 分析步骤定义对象 | `ProcedureRuntime` |
| `Compiler` | 编译层 | 开发为主 | 把定义对象编译为运行时对象图的服务类 | `PyFEMSession` |
| `DofManager` | 编译层 | 开发为主 | 正式自由度注册、编号和冻结对象 | `Assembler` |

#### 2.2 Kernel runtime 家族

| 术语 | 所属层 | 面向对象 | 推荐解释 | 容易和谁混淆 |
| --- | --- | --- | --- | --- |
| `MaterialRuntime` | kernel 层 | 开发为主 | 材料运行时根接口，负责材料更新与材料状态 | `MaterialDef` |
| `SectionRuntime` | kernel 层 | 开发为主 | 截面运行时根接口，负责截面语义与材料绑定 | `SectionDef` |
| `ElementRuntime` | kernel 层 | 开发为主 | 单元运行时根接口，负责局部刚度、残量、质量和输出 | `Assembler` |
| `ConstraintRuntime` | kernel 层 | 开发为主 | 约束运行时根接口，负责受约束 DOF 集合 | `BoundaryDef` |
| `InteractionRuntime` | kernel 层 | 开发为主 | 相互作用运行时根接口 | `InteractionDef` |
| provider | 编译 / kernel 边界 | 开发为主 | 把定义对象构造成正式 runtime 的提供者 | runtime 本体 |
| placeholder runtime | 编译 / kernel 边界 | 开发为主 | provider 缺失时的受控占位对象 | 正式支持能力 |

#### 2.3 Procedures 家族

| 术语 | 所属层 | 面向对象 | 推荐解释 | 容易和谁混淆 |
| --- | --- | --- | --- | --- |
| `ProcedureRuntime` | procedures 层 | 开发为主 | 过程运行时根接口，负责执行一步分析 | `StepDef` |
| `StepProcedureRuntime` | procedures 层 | 开发为主 | 基于步骤的过程运行时基类 | `DiscreteProblem` |
| procedure provider | 编译 / procedures 边界 | 开发为主 | 把 `procedure_type` 绑定到具体过程实现的提供者 | 过程对象本体 |
| `static_linear` / `static_nonlinear` / `modal` / `implicit_dynamic` | procedures 层 | 用户 + 开发 | 当前正式过程家族键 | GUI 预留入口中的未来过程 |
| `state transfer channel` | procedures / solver 边界 | 开发为主 | 多步状态继承使用的通道名 | 结果历史量名 |

#### 2.4 Solver 家族

| 术语 | 所属层 | 面向对象 | 推荐解释 | 容易和谁混淆 |
| --- | --- | --- | --- | --- |
| `Assembler` | solver 层 | 开发为主 | 全局装配服务对象 | `ElementRuntime` |
| `DiscreteProblem` | solver 层 | 开发为主 | 包装装配、状态与约束处理的离散问题对象 | `ProcedureRuntime` |
| `LinearAlgebraBackend` | solver 层 | 开发为主 | 线性系统与特征值求解后端接口 | `Assembler` |
| `SciPyBackend` | solver 层 | 开发为主 | 当前正式内置线性代数后端实现 | `PyFEMSession` |
| `RuntimeState` | solver 层 | 开发为主 | 求解期间的内部状态对象 | `ResultFrame` |
| `StateManager` | solver 层 | 开发为主 | 分配、复制、提交和回滚运行时状态 | `ResultsWriter` |
| `ReducedSystem` | solver 层 | 开发为主 | 约化系统矩阵与自由索引容器 | 完整求解器对象 |

#### 2.5 结果与消费家族

| 术语 | 所属层 | 面向对象 | 推荐解释 | 容易和谁混淆 |
| --- | --- | --- | --- | --- |
| `ResultsSession` | 结果层 | 用户 + 开发 | 一次结果写出 / 读取会话的元数据 | `ResultsFacade` |
| `ResultsDatabase` | 结果层 | 用户 + 开发 | 按步骤组织的正式结果数据库对象 | `ModelDB` |
| `ResultStep` | 结果层 | 用户 + 开发 | 一个步骤的全部结果 | `StepDef` |
| `ResultFrame` | 结果层 | 用户 + 开发 | 单帧场结果对象 | `RuntimeState` |
| `ResultField` | 结果层 | 用户 + 开发 | 单个场变量对象 | probe 序列 |
| `ResultHistorySeries` | 结果层 | 用户 + 开发 | 历史量序列对象 | `ProbeSeries` |
| `ResultSummary` | 结果层 | 用户 + 开发 | 步骤级摘要对象 | 控制台日志 |
| `ResultsFacade` | 结果层消费侧 | 用户 + 开发 | reader-only 结果浏览入口 | `ResultsDatabase` |
| `ResultsQueryService` | 结果层消费侧 | 用户 + 开发 | facade 派生的筛选与概览服务 | `ResultsProbeService` |
| `ResultsProbeService` | 结果层消费侧 | 用户 + 开发 | facade 派生的探针与 CSV 导出服务 | `ResultsQueryService` |

#### 2.6 Job / API / GUI 壳层家族

| 术语 | 所属层 | 面向对象 | 推荐解释 | 容易和谁混淆 |
| --- | --- | --- | --- | --- |
| `PyFEMSession` | 产品壳层 | 用户 + 开发 | Python API 的统一会话入口 | `Compiler` |
| `JobManager` | 产品壳层 | 开发为主 | 标准作业执行外壳 | `ProcedureRuntime` |
| `JobExecutionReport` | 产品壳层 | 用户 + 开发 | 作业执行结果摘要对象 | `ResultSummary` |
| `GuiShell` | GUI 壳层 | 用户 + 开发 | GUI 工作台的运行与结果入口 | `PyFEMSession` |
| `GuiResultsViewContext` | GUI 壳层 | 开发为主 | GUI 结果视图的消费上下文 | `ResultsDatabase` |
| VTK exporter | 产品壳层 / 导出边界 | 用户 + 开发 | 消费结果对象并导出 VTK 的壳层服务 | 结果层本体 |

### 3. 易混淆对象对照表

| 对象对 | 当前正确区分 |
| --- | --- |
| `ModelDB` vs `CompiledModel` | 前者是定义对象图，后者是编译产物与状态快照持有者 |
| `StepDef` vs `ProcedureRuntime` | 前者定义步骤，后者执行步骤 |
| `MaterialDef` vs `MaterialRuntime` | 前者是定义对象，后者是编译后的材料运行时对象 |
| `ElementRuntime` vs `Assembler` | 前者提供局部贡献，后者装配全局系统 |
| `ProcedureRuntime` vs `DiscreteProblem` | 前者编排步骤，后者包装离散求解问题 |
| `RuntimeState` vs `ResultFrame` | 前者是内部求解状态，后者是正式结果帧 |
| `ResultsSession` vs `ResultsDatabase` | 前者是会话元数据，后者是完整结果层级 |
| `ResultsDatabase` vs `ResultsFacade` | 前者是结果本体，后者是 reader-only 门面 |
| `ResultsQueryService` vs `ResultsProbeService` | 前者做筛选与概览，后者做序列抽取与导出 |
| `PyFEMSession` vs `JobManager` | 前者是用户入口，后者是内部作业外壳 |

### 4. 易混淆方法与入口

| 方法或入口 | 当前正确区分 |
| --- | --- |
| `load_model_from_file()` vs `run_input_file()` | 前者只读模，后者继续执行编译、求解和结果写出 |
| `run_input_file()` vs `run_model()` | 前者以输入文件为来源并可推导默认路径，后者以既有 `ModelDB` 为来源 |
| `open_results()` vs `ResultsFacade.field()` | 前者打开结果门面，后者从门面读取字段对象 |
| `ResultsFacade.query()` vs `ResultsFacade.probe()` | 前者下钻到筛选服务，后者下钻到探针服务 |
| `Compiler.compile()` vs `CompiledModel.get_step_runtime()` | 前者构建编译产物，后者从编译产物中读取过程对象 |
| `ResultsQueryService.history()` vs `ResultsProbeService.history()` | 前者返回 `ResultHistorySeries`，后者返回可直接消费的 probe 序列 |

### 5. 术语使用约定

| 约定 | 说明 |
| --- | --- |
| 用户章节优先中文主叫法 | 例如“模型数据库”“结果门面”“分析步骤” |
| 开发章节优先分层术语 | 优先写“定义层 / 编译层 / kernel / procedures / solver / results / 壳层” |
| `kernel / procedures / solver` 必须分开使用 | 不再回到“运行时层大桶”口径 |
| `ResultsFacade`、`ResultsDatabase`、`ResultsSession` 必须严格区分 | 避免结果本体与消费入口混称 |
| placeholder 不进入当前支持口径 | 保持支持边界保守 |
| 证据不足时明确写“待确认” | 不把推测写成已支持 |

### 6. 本章用途

本章不是附录式翻译表，而是全文回查基线。继续维护总稿时，应优先用它检查：

- 同一对象是否在不同章节被写成不同层次。
- `kernel / procedures / solver` 是否又被混回一层。
- `ResultsFacade` 是否被误写成数据库对象。
- `RuntimeState` 是否被误写成结果对象。
- placeholder、测试对象或预留入口是否被误写成正式支持。
