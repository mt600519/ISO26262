# VC‑Z01X 故障仿真（Fault Simulation）完整学习手册

> **用途**：车规 ISP 芯片验证 · ISO 26262 审查（Assessment）准备
> **依据**：Synopsys《VC Z01X User Guide, X‑2025.06》《Z01X Manufacturing Assurance User Guide, X‑2025.06》
> **说明**：文中页码均指上述两份手册；术语与语法最终以你所用版本的 User Guide 正文与附录 A/B 为准。

| 文档信息 | 内容 |
|----------|------|
| 覆盖范围 | 常用 fault simulation 概念、机制、分类、覆盖率、流程、陷阱、审查问答 |
| 适用场景 | 功能安全故障仿真（SPFM / LFM / DC 量化），非制造测试（DFT/ATPG）为主 |
| 关键工具 | VC Z01X 编译器、vc_fcc、vc_fcm、Z01X 引擎、FDB、SFF |
| 版本基准 | X‑2025.06 |

---

## 目录

| 章 | 标题 | 一句话 |
|----|------|--------|
| 0 | 为什么做故障仿真 | ISO 26262 指标与故障仿真的关系 |
| 1 | 工具链与数据流 | 各组件职责与数据走向 |
| 2 | 故障模型 | Stuck‑At / Transient / Transition / Bridge |
| 3 | 注入与观测机制 | GM/FM、Observe vs Detect、UCFD 系统任务 |
| 4 | 故障分类系统 | 两字符状态、内建组、Dropping、UDFS 映射 |
| 5 | 覆盖率计算 | Test/Fault Coverage、DC、SPFM/LFM |
| 6 | SFF 标准故障格式 | 各段用途与语法 |
| 7 | FCM 常用命令 | vc_fcm 命令与脚本 |
| 8 | 完整流程 | 官方流程 + 从零搭建 step‑by‑step |
| 9 | 常见陷阱与审查红线 | 最易出错、最易被追问的点 |
| 10 | 工具置信度与 CoU/AoU | Tool qualification 与强制条款 |
| 11 | 审查 Q&A 速记 | 评估方高频问题与应答 |
| 12 | 术语对照表 | 中英文术语速查 |

---

## 0. 为什么做故障仿真（ISO 26262 背景）

功能验证回答"设计对不对"；**故障仿真回答另一个问题：当硬件里真的发生随机故障时，
安全机制（Safety Mechanism, SM）能不能发现/纠正它？**

ISO 26262 用三个硬件指标量化，故障仿真给出的**诊断覆盖率（Diagnostic Coverage, DC）**是它们的核心输入：

| 指标 | 全称 | 关注的故障 |
|------|------|-----------|
| **SPFM** | Single‑Point Fault Metric（单点故障度量） | 单点 + 残余故障是否被 SM 覆盖 |
| **LFM** | Latent‑Fault Metric（潜伏故障度量） | 潜伏的多点故障是否被第二重手段覆盖 |
| **PMHF** | Probabilistic Metric for random HW Failures | 随机硬件失效的整体概率 |

> **一句话**：SPFM/LFM 的分子分母 = "被检测的危险故障" 与 "全部危险故障" 之比；
> 要得到这个比值，就必须逐个注入故障、仿真、判定检测状态——这正是 VC‑Z01X 的职责。

---

## 1. 工具链与数据流

### 1.1 组件职责

| 组件 | 职责 | 产物/接口 |
|------|------|-----------|
| **VC Z01X 编译器**（基于 VCS） | 把 DUT 编译成可注入故障的模型 | `simv` + `simv.daidir` |
| **vc_fcc**（Fault Campaign Compiler） | 生成故障清单、完成塌缩 | 写入 FDB |
| **vc_fcm**（Fault Campaign Manager，FCM） | 总控：建 campaign、派测试、调度集群、跑仿真、出报告 | 交互/脚本 |
| **Z01X 并发引擎（COATS）** | 参考机与多个故障机并发推进并比较 | — |
| **FDB**（Fault Database） | 故障与结果数据库 | `fdb_connect` 连接 |
| **SFF**（Standard Fault Format） | 贯穿全流程的文本文件 | 既是输入也是输出 |

### 1.2 数据流

```
设计(RTL/门级) + TB/激励
      │  VC Z01X 编译
      ▼
 simv + simv.daidir ──► 先跑一次"无故障"逻辑仿真验证
      │  vc_fcc（读 SFF / 自动生成）
      ▼
 Fault Universe（写入 FDB，已塌缩）
      │  vc_fcm: create_campaign → create_testcases → fsim
      ▼
 并发故障仿真（GM vs 多个 FM）
      │  report
      ▼
 覆盖率报告（SFF 格式）──► 分析漏检 → 迭代收敛
```

---

## 2. 故障模型（Fault Models）

### 2.1 基本概念

| 概念 | 定义 |
|------|------|
| **故障（Fault）** | 物理缺陷在电路层面的抽象；仿真不模拟物理机理，而是把缺陷抽象为"某信号/端口呈现错误逻辑行为" |
| **故障模型（Fault Model）** | 规定"如何抽象缺陷"的规则 |

### 2.2 四种故障模型总览（先分清用途）

Z01X 的故障模型分两类用途——**功能安全（ISO 26262）** 与 **制造测试（DFT/生产）**（出处：UG p50/p81；Mfg Guide 第 9 章）：

| 故障模型 | 用途 | 建模的缺陷 | 放置位置 | 子类型 |
|----------|------|-----------|----------|--------|
| **Stuck‑At（固定）** | **功能安全** | 永久缺陷（短路、开路、tie 常量） | primitive 端子/模块端口/net/array | sa0、sa1（6 类位置） |
| **Transient（瞬态/软错误）** | **功能安全** | SEU 软错误（粒子翻转，瞬时） | **时序元件输出、存储器** | 发生一次即消失 |
| Transition（时延/跳变） | 制造测试 | 跳变过慢 | wire/primitive/port | slow‑to‑rise(R)、slow‑to‑fall(F) |
| Bridge（桥接） | 制造测试 | 两 net 短接 | 一对 net | wire、wor、wand |

> **对车规 ISP 功能安全而言，真正相关的是 Stuck‑At + Transient 两种。**
> Transition 与 Bridge 属制造测试（DFT/ATPG），一般不作为 SPFM/LFM 主证据。

### 2.3 Stuck‑At（固定型）——功能安全主力

在**整个仿真期间**把 gate 的输入/输出/net **永久钉死在 0（sa0）或 1（sa1）**，建模永久物理缺陷。

| 项 | 内容 |
|----|------|
| 6 类位置 | input sa0/sa1、output sa0/sa1、net sa0/sa1 |
| Single stuck‑at | 一次钉死一个位置，由 vc_fcc 自动生成，**功能安全主力** |
| Multipoint stuck‑at | 同时钉死多个位置，只能经 SFF 手工/脚本描述，位置间用 `+` 连接 |

### 2.4 Transient（瞬态/软错误）——ISO 26262 的另一半

**只发生一次、随即消失**的故障（原文 "faults that occur once and then disappear"），建模 **软错误 / SEU**。

| 项 | 内容 |
|----|------|
| 放置限制 | 只能放在**时序元件（触发器/寄存器）输出**和**存储器（memory）**上 |
| 对 ISP 的意义 | line/frame buffer、寄存器、SRAM 的软错误靠 **ECC/parity** 覆盖 |
| 支撑的指标 | 量化 SM 对软错误的诊断覆盖 + 评估**潜伏故障 LFM**（仅做 stuck‑at 无法回答） |

### 2.5 Transition（时延）——制造测试

建模信号**跳变过慢**：slow‑to‑rise(R, 0→1 变慢)、slow‑to‑fall(F, 1→0 变慢)，放 wire/primitive/port。

> **必要条件（常被追问）**：同一故障点若连 stuck‑at 都测不到，则其 transition 也测不到，
> 故 transition 流程以 stuck‑at 的 detected+potential 清单作**起始清单**。
> 出处：Mfg Guide p172 —*"If a fault is unable to be detected as a stuck‑at fault, it is impossible to detect it as a transition fault."*；UG p284 佐证。
> 这是**可测性（testability）层面**的必要条件，针对同一故障点，非指某次具体仿真结果。

### 2.6 故障位置对象与塌缩

**故障位置对象（Fault Location Types，支持通配）**：

| 关键字 | 含义 | 关键字 | 含义 |
|--------|------|--------|------|
| `PORT` | 端口 | `ARRY` | 数组 |
| `PRIM` | primitive 端子 | `VARI` | 变量 |
| `WIRE`/`NET` | 连线 | `FLOP` | 触发器 |

**故障塌缩（Fault Collapsing）与 Prime Fault**：

| 项 | 说明 |
|----|------|
| 定义 | 逻辑**完全等价**的故障合并，只保留代表——**prime（源）故障** |
| 被合并者 | 标记 `--`（Collapsed），状态跟随 prime |
| 目的 | 只仿真 prime，避免重复计算，大幅缩短时间 |
| 报告 | 默认不单列 `--`（除非 `+fault+status+--`），但计入 "All" 列；分 **Prime / Total** 两套统计（见 §5.4） |

---

## 3. 故障注入与观测机制

> ⚠️ 本章是审查最易深挖、也最易出错（导致覆盖率虚高）的部分，务必吃透。

### 3.1 参考机与故障机 · 检测判据

| 概念 | 说明 |
|------|------|
| **Good Machine（GM，参考机）** | 不注入任何故障的"黄金"实例 |
| **Faulty Machine（FM，故障机）** | 注入了某个（组）故障的实例 |
| 并发仿真 | Z01X 让 GM 与多个 FM **同时推进**，在观测点比较——比串行快得多 |
| **检测判据** | 在被观测位置与时刻，**FM 处于一个已知态、GM 处于另一个不同已知态**，即判为"检测" |
| X 态处理 | GM 已知而 FM=X（或反之）通常**不算检测**，可能计 potential(PT)；Z01X 对 X **保守**，不高估覆盖 |

### 3.2 观测点 vs 诊断点（Observe vs Detect）——两套独立机制

Z01X 把"故障暴露"拆成两层，对应 ISO 26262 的两个问题：

| 机制 | SFF 段 | 系统函数 | 回答的问题 | 放置位置 |
|------|--------|----------|-----------|----------|
| **观测（Observability）** | `Observe`（在 FailureMode 中） | `$fs_observe` | 故障是否传播到安全相关输出（**是否"危险"**） | 安全相关输出引脚 |
| **诊断（Detection）** | `Detect` / `SafetyMechanism` | `$fs_detect` | 故障是否被安全机制捕获（**是否"被诊断"**） | SM 检测输出（ECC flag / lockstep / CRC / STL 报警） |

> **交叉两者**才能区分：危险且被诊断（good）／危险但未被诊断（漏网）／未传播到输出（安全）。
> `$fs_observe` **只在 FM 中有效**（在 GM 中返回 0）；观测点由该 FM 关联的 **FailureMode** 决定。

### 3.3 UCFD 系统任务（User Controlled Fault Detection）

用于自定义观测/检测行为的系统任务（UG p251）。**只在 FM 生效**，不改变参考仿真，若在 GM 执行会告警：

| 系统任务/函数 | 作用 |
|---------------|------|
| `$fs_compare` | 比较 GM/FM 指定信号；工具自动按 `#0 … #0` 同步，避免竞争 |
| `$fs_observe` | 比较 SFF `Observe` 块信号（观测点）；只在 FM 有效 |
| `$fs_detect` | 比较 SFF `Detect` 块信号（诊断点） |
| `$fs_strobe` | 在**指定时刻**比较列出的选通点 |
| `$fs_strobe_onevent()` | 信号**每次变化时**比较（持续监视，无需给定时刻） |
| `$fs_set_status` / `$fs_drop_status` | 手工设置 / 丢弃某故障状态 |

### 3.4 Strobe 的两种定义方式

| 方式 | 做法 | 特点 |
|------|------|------|
| 改设计 | TB/SM 中插 `$fs_strobe` 或 `$fs_strobe_onevent`；可指定 net/变量/模块实例/interface（指定模块实例=其所有输出端口） | 灵活，需改代码 |
| 不改设计 | SFF 写 `Strobe` 段声明信号（等价 `$fs_strobe_onevent()`） | 无需改 TB |

### 3.5 最佳实践与陷阱（审查红线）

| 陷阱 / 实践 | 说明 | 出处 |
|-----------|------|------|
| **observe+detect 写在同一 procedural block** | 拆成两个独立 `always` 块 → 执行顺序不定 → 故障结果**非确定性** | UG p253 |
| **strobe 集必须覆盖所有可观测位置** | 端口不在 strobe 列表 → 故障被标 **UU**，默认**不计未检测** → 覆盖率**虚高** | Mfg Guide p195 |
| strobe 只在 FM 生效 | 在 GM 执行报 warning，属正常 | UG p251 |
| 无 strobe 的测试会被移出 | 日志 `Error: Test t1 contains no strobes` 必须处理，不能忽略 | Mfg Guide p324 |

推荐写法：
```verilog
always @(negedge clk) begin
   cmp = $fs_compare(observe_signal);
   if (cmp == /*已观测*/) ... else $fs_drop_status("ND", detect_signal);
end
```

---

## 4. 故障分类系统（Fault Status）

### 4.1 两字符状态码：观测 × 检测

一个故障类用**两个字符**表示（UG p257）：

| 位置 | 含义 |
|------|------|
| 第 1 字符 | **观测状态（observation）**：是否传播到观测点 |
| 第 2 字符 | **检测状态（detection）**：是否被安全机制检测到 |

> 状态码不区分大小写、默认输出大写，每个码带引号描述串。可组成 **status group**（两字符缩写+组名+成员）用于报告。
> 例：`OD "Observed Diagnosed"`（已观测 且 已诊断）。

### 4.2 内建故障状态组

| 内建组 | 含义 | 代表成员 |
|--------|------|----------|
| **DG（Detected）** | 已检测 | DT、DD、DE、DF、DO |
| **Potential** | 潜在检测（含 X 不确定） | PT、PD |
| **Untestable（UG）** | 结构不可测 = 功能安全里的**安全故障** | UT、UU、UB、UR、UI、UO |
| **Not Detected** | 未检测 | ND、NC、NN、NO |
| Hyper / Oscillating | 超故障 / 振荡故障 | HA、OZ |
| Collapsed | 塌缩（跟随 prime） | `--` |

### 4.3 关键状态详解

**Detected 组**：

| 状态 | 含义 |
|------|------|
| `DT`（Detected） | 被检测**至少一次、未达最大次数** |
| `DD`（**Dropped Detected**） | 被检测**达最大次数**后丢弃；阈值 `-fsim=maxhard:<num>`，**默认 1** |
| `DE`（Detected Error） | 故障机触发 assertion → `$fatal`/`$error` |
| `DF` / `DO` | 其他已检测细分（`DO` 由 `$fs_set_status`/`$fs_drop_status` 赋值） |

> ⚠️ **命名陷阱**：`DD` 的**内建**含义是 **"Dropped Detected"**；但功能安全 **UDFS** 里用户常把 `DD`
> **重定义**为 **"Dangerous Diagnosed"**。二者完全不同，只是复用同一两字符码——出示 SFF 时务必讲清用的是哪个。

**Potential 组**：

| 状态 | 含义 |
|------|------|
| `PT`（Potential） | X 值导致"可能被检测"的不确定情形；覆盖率通常按 **0.5 权重**折算 |
| `PD`（Dropped Potential） | 达丢弃阈值的 potential 故障 |

**Untestable 组（= 功能安全里的 Safe fault，UG p62/p271，Mfg Guide p150）**：

> 定义：**结构上不可能造成器件失效、与激励无关**的故障（连电源/地、tie 常量、网表未连接）。功能安全里就是**安全故障**。

| 状态 | 含义 | 覆盖统计 |
|------|------|----------|
| `UT`（Untestable Tied） | 连到 supply net 或常量（tie off） | 默认**不计入** |
| `UU`（Untestable Unused） | 未使用/未观测（← §3.5 虚高陷阱） | 默认**不计入** |
| `UB`（Untestable Blocked） | 被连到 supply/常量的信号阻断 | — |
| `UR`（Untestable Redundant） | 被另一故障屏蔽（冗余） | — |
| `UI`（Untestable Imported） | 导入清单时无法判因，仅 import 时赋值 | — |
| `UO` | 其他不可测类别 | — |

**Not Detected 组**：`ND`（未检测，含 GM=X、FM=known）、`NC`/`NN`/`NO` 等"未观测/未检测"组合。

**特殊故障**：

| 状态 | 含义 | 处理 |
|------|------|------|
| `HA`（Hyperfault，超故障） | 引发海量事件、极慢/耗内存（破坏时钟/复位网络） | **默认移除**（建议保留此默认），**按危险故障计入覆盖**；用 Resolving Hyperfaults 单独判定（UG p276） |
| `OZ`（Oscillating，振荡） | 信号无休止振荡 | 优先级高于 PT，判定后不再反复仿真 |

> HA/OZ 优先级**高于** potential(PT)——副作用是 potential 覆盖可能在测试间下降。

### 4.4 Fault Dropping（故障丢弃）

| 项 | 说明 |
|----|------|
| 触发 | 故障被检测达 `-fsim=maxhard:<num>`（默认 1）次数，状态 `DT`→`DD`，**不再重复仿真** |
| 收益 | 大幅加速回归 |
| 注意 | 要评估**多个** SM 各自覆盖分布时，需保留故障（"Sharing Results Between Failure Modes"），不能过早 drop |

### 4.5 UDFS 与 ISO 26262 映射

**UDFS（User‑Defined Fault Status）**：用户在 SFF 里重定义/新增两字符状态与分组。
**复用任何内建两字符码前，必须先重定义该内建状态**并声明交互方式（UG p257）。典型功能安全 UDFS：

```
StatusDefinition {
    DD "Dangerous Diagnosed"      (ND, NP, OD);   // 危险—被 SM 诊断（注意 DD 已被重定义！）
    DN "Dangerous Not Diagnosed"  (ON, OP, PN);   // 危险—未被诊断（残余/潜伏）
    SA "Safe"                     (UB, UR, UT, UU);// 安全故障（Untestable + 结构安全）
    SU "Dangerous Unobserved"     (NC, NN, NO, NT);
}
Coverage {
    "Diagnostic Coverage" = "DD / (NA + DA + DN + DD)";
}
```

映射到 ISO 26262（帮助向审查方解释）：

| ISO 26262 概念 | 在 Z01X 里的体现 |
|----------------|------------------|
| Safe fault（安全故障） | Untestable（UT/UU/UB/UR）+ 结构安全 → `Safe` 组 |
| Single‑Point / Residual Fault（未被 SM 覆盖） | 危险且未被诊断 → `DN` |
| SPF/RF 中被 SM 覆盖的部分 | 危险且被诊断 → `DD`（此处 = Dangerous Diagnosed） |
| Diagnostic Coverage (DC) | `DD /（全部危险故障）` 覆盖方程 |
| Latent / Multi‑point Fault（LFM） | Transient + 第二重 SM 的观测/诊断点定义相应组 |

---

## 5. 覆盖率计算

### 5.1 两个内建覆盖率

| 覆盖率 | 公式 | 分母是否含 Untestable |
|--------|------|----------------------|
| **Test Coverage** | `Weighted_Total / Total` | **排除** |
| **Fault Coverage** | `Weighted_Total / (Total + UB + UT + UU + UR + UO + UI)` | **包含** |

出处：UG p216/p266，Mfg Guide p390。

### 5.2 Weighted_Total 与权重

`Weighted_Total = Σ(各状态计数 × 权重)`，例：`DD×w + DT×w + PD×w + PT×w + …`

| 状态 | 典型权重 |
|------|----------|
| 已检测（DD/DT） | 1 |
| potential（PT/PD） | **0.5**（半计，体现不确定性） |

### 5.3 Diagnostic Coverage / SPFM / LFM

功能安全用的 **DC 是用户在 SFF Coverage 段自定义**的方程（非上面两个内建式），一般 `DD /（全部危险）`，进一步喂：

| ISO 26262 指标 | 近似关系 |
|----------------|----------|
| **SPFM** | ≈ 1 − Σ(残余故障失效率) / Σ(安全相关故障失效率) |
| **LFM** | ≈ 1 − Σ(潜伏故障失效率) / Σ(可能潜伏的多点故障失效率) |

> **审查关键**：Coverage 方程**可配置**，必被问"如何定义、如何对应 ISO 26262‑5"。SFF Coverage 段要有注释+评审记录。

### 5.4 Prime vs Total

| 列 | 含义 | 功能安全用法 |
|----|------|-------------|
| **Prime** | 去等价/塌缩后的计数 | 指标一般基于此 |
| **Total** | 含塌缩故障的计数 | `--` 计入 "All"，默认不单列 |

---

## 6. SFF 标准故障格式详解

SFF 是贯穿输入输出的核心文本文件，审查必看（UG 第 9 章 / Mfg Guide 第 10 章）：

| 段（Section） | 用途 | 备注 |
|---------------|------|------|
| Block / Test Information | 文件头、块与测试信息 | — |
| **Fault Status Definition** | 定义/重定义两字符状态与分组（UDFS） | 功能安全语义在此 |
| **Coverage** | 定义覆盖率方程（含 DC） | 用 UDFS 时**必须**提供 |
| **Strobe** | 声明选通信号（等价 `$fs_strobe_onevent`） | 无需改 TB |
| StrobeData | 报告用：故障命中时显示的引脚状态 | 仅输出 |
| Constraint | 约束信号值 | 值**只能 0 或 1**，不支持多 bit |
| FaultGenerate | 定义在哪些位置**生成**故障（支持通配） | 输入 |
| FaultInfo | 故障附加信息 | — |
| **FaultList** | 逐条故障描述 | 见下 |
| **FailureMode** | 把 FMEDA 失效模式与观测信号+SM 关联 | `Observe` 必需；`Detect` 或 `SafetyMechanism` 必需 |
| **SafetyMechanism** | 定义 SM 检测信号列表 | 配合 `$fs_detect`；可命名复用 |
| Promotion（子块） | 检测状态提升规则（potential→detected） | — |

**FaultList 语法**：`<fault info> <status> <fault type> {<location>} [results]`
```
<  1> OD 0 {PORT "test.risc1.alu1.accum[0]"}
      -- 0 {PORT "test.risc1.alureg.d1.q"}    // 塌缩到上面的 prime
```

**SafetyMechanism 语法**：
```
SafetyMechanism sm_lockstep {
    Detect { "top.dut.cpu.compare_error" }
}
```

---

## 7. FCM 常用命令

| 命令 | 作用 |
|------|------|
| `create_campaign` | 定义并创建故障宇宙（读 SFF、指定 DUT、daidir） |
| `create_testcases` | 定义测试用例（可执行、参数、UVM testname） |
| `fdb_connect` | 连接 FDB（`fsim` 前提） |
| `set_campaign` / `set_config` | 选定当前 campaign / 设置运行配置 |
| `fsim` | 执行故障仿真，自动管理集群作业与重试 |
| `report` | 生成覆盖率报告（可随时出） |
| `save_session` | 保存会话（断点续跑） |
| `quit` / `exit` | 退出 FCM |

**脚本示例**：
```tcl
create_campaign -args "-full64 -daidir simv.daidir \
                       -campaign fc1 -sff user.sff -dut test.dut"
create_testcases -name test1 -exec "$::env(PWD)/simv" \
                 -args "+UVM_TESTNAME=test1"
catch {fsim}
report -campaign fc1 -report user_coverage.sff -overwrite
```

| 用途 | 选项 |
|------|------|
| 分布式调度 | `-max_faults_per_task`、`-localhost`、作业失败自动 retry |
| 调试发散输出 | `create_testcases -fsim_args -fsim=fault+messages+all`（发散行打 `<FID>` 标签） |
| 完整语法 | 附录 A（FCM Commands）/ 附录 B（Command Syntax） |

---

## 8. 完整流程

### 8.1 官方程序化流程（UG p50 / p183）

| 步 | 动作 | 产物/目的 |
|----|------|-----------|
| 1 | 编译并"无故障"逻辑仿真 | 验证编译正确，生成 `simv` + `simv.daidir` |
| 2 | 生成故障清单 | 一次性设定故障生成/导入选项 |
| 3 | FCM 建 campaign、派测试、跑仿真 | 得到故障状态 |
| 4 | 出报告、分析、迭代 | 收敛 DC |

### 8.2 并发单遍 FMEDA 流程（推荐，UG p274）

**一次故障注入运行**即产出"哪些危险故障被 SM 检测到"的完整报告，直接喂 FMEDA。
前提：提前在 TB/SM 里定义好观测点（Observe）与诊断点（Detect），FCM + SFF + Z01X 协同一遍跑完分类。

### 8.3 从零搭建 Step‑by‑Step

| 阶段 | 做什么 | 关键点 |
|------|--------|--------|
| **0 备输入** | 设计（门级网表+库）、激励（UVM/VCD/forcelist）、SM 检测信号清单、FMEDA 初稿、集群 | 功能安全 SPFM/LFM 通常基于门级 |
| **1 编译+验证好机** | VC Z01X 编译产 `simv`/`daidir`；**先跑无故障逻辑仿真** | `-fsim=dut:<层次>` 阻止故障传播到 DUT 外；穿 class 加 `-fsim=class` |
| **2 定义观测/诊断点** | Observe 放安全相关输出；Detect 放 SM 检测输出；或 SFF `Strobe` 段 | observe+detect 同一 block；strobe 覆盖所有可观测位置（防虚高） |
| **3 生成故障清单** | `vc_fcc` 生成 fault universe、塌缩、写 FDB | 可选结构分析预识别 Untestable/安全故障 |
| **4 编写 SFF** | UDFS 状态分组、Coverage 方程、SafetyMechanism/FailureMode | DC 方程加注释对应 ISO 26262‑5；注意 DD 重定义 |
| **5 建 campaign** | `create_campaign` → `create_testcases` | 见 §7 脚本 |
| **6 跑故障仿真** | `catch {fsim}` | 分布式并发 |
| **7 出报告+迭代** | `report`；盯 **DN**（危险未诊断）列 | 增强 SM 观测点（参考 Observability Targeted Report）/补激励/论证 safe，回阶段 2/5 迭代 |
| **8 归档** | 脚本、SFF、种子、`simv`、报告、版本号；`save_session` 续跑 | 完成 CoU/AoU 检查表 |

**一页流程图**：
```
[0] 备输入 → [1] 编译(-fsim=dut)+无故障验证 → [2] Observe+Detect(同一block;strobe全覆盖)
   → [3] vc_fcc 故障清单+塌缩 → [4] SFF(UDFS+Coverage+SM) → [5] create_campaign/testcases
   → [6] fsim(并发) → [7] report→分析DN→补SM/激励→(回[2]/[5]迭代) → [8] 归档+CoU/AoU
```

### 8.4 官方已验证的示例流程

| 流程 | 特点 |
|------|------|
| **Concurrent（并发，推荐）** | 单遍产出 FMEDA 数据 |
| Serial（串行） | 更线性，便于排查 |
| Using Software Test Libraries（STL） | 面向带软件自检的场景 |
| Resolving Hyperfaults | 隔离并单独判定 HA/OZ |

---

## 9. 常见陷阱与审查红线

| # | 陷阱 | 后果 | 应对 | 出处 |
|---|------|------|------|------|
| 1 | strobe 集不完整 | 故障标 UU、默认不计未检测 → 覆盖率**虚高** | 生成故障含所有可观测位置；核查 UU 占比 | Mfg p195 |
| 2 | observe/detect 拆多个 always 块 | 结果**非确定** | 写在同一 procedural block | UG p253 |
| 3 | `DD` 双重含义 | 审查误解 | 内建=Dropped Detected；UDFS=Dangerous Diagnosed，出示时讲清 | UG p60/p273 |
| 4 | Untestable 默认不计覆盖 | 安全故障未论证 | 当"安全故障"须有结构论证，不只靠工具默认 | UG p271 |
| 5 | Hyperfault 默认移除但按危险计 | 当作已解决 | 用 Resolving Hyperfaults 单独判定 | UG p276 |
| 6 | X 态处理 | 高估覆盖 | 保守计未检测/potential；PT 按 0.5 折算 | UG p52 |
| 7 | 忽略日志 Warning/Error | 违反 CoU；测试被静默移除 | 逐条审阅所有消息 | §10 / Mfg p324 |
| 8 | Fault dropping 阈值 | 多 SM 覆盖评估失真 | `-fsim=maxhard` 默认 1，不要过早 drop | UG p52 |

---

## 10. 工具置信度与 CoU/AoU

| 项 | 内容 |
|----|------|
| **Tool Qualification** | ISO 26262‑8 Clause 11 要求评估工具置信度（TCL）；出示 Synopsys VC‑Z01X 功能安全认证 / qualification kit |
| **CoU（Conditions of Use）** | 载于《VC Z01X Functional Safety Manual》，强制遵守 |
| **AoU（Assumptions of Use）** | 同上 |
| 典型条款 | ① **审阅 log 里所有 Info/Warning/Error/Fatal**；② 遵守 Z01X 功能安全指南；③ 用官方验证流程（Concurrent/Serial/STL） |
| 审查准备 | 实际流程、SFF 定义、Coverage 方程、strobe 完整性证据、CoU/AoU 符合性逐条留档 |

---

## 11. 审查 Q&A 速记

| # | 问题 | 应答要点 |
|---|------|----------|
| 1 | 用什么故障模型？为什么？ | 功能安全用 **Stuck‑At + Transient**：SA 对应永久故障（6 类位置，支撑 SPFM）；transient 对应 SEU 软错误（放 FF 输出与存储器），评估 ECC/parity 对 LFM 的覆盖。Transition/Bridge 属制造测试。 |
| 2 | 全故障还是抽样？代表性？ | vc_fcc 基于门级网表全量生成+塌缩；如抽样，给方法、置信区间、样本量依据。 |
| 3 | 安全故障如何识别论证？ | Untestable（tie/unused/redundant/blocked）+ 结构安全；每类留工程论证，不只靠工具默认。 |
| 4 | DC 方程如何定义？对应 SPFM/LFM？ | 出示 SFF Coverage 段（`DD/(全部危险)`），解释 DD=Dangerous Diagnosed 如何进入 SPFM=1−ΣRF/Σ(SPF+RF)。 |
| 5 | 观测点/诊断点如何区分与保全覆盖？ | Observe=安全相关输出，Detect=SM 检测输出；strobe 覆盖所有可观测位置，出示 Observability Targeted Report。 |
| 6 | X 态会否高估覆盖？ | 保守计未检测/potential，PT 按 0.5 折算；说明检测判据。 |
| 7 | Hyperfault/振荡怎么处理？ | 默认移除但按危险计；用 Resolving Hyperfaults 单独判定，未静默丢弃。 |
| 8 | strobe 虚高如何避免？ | 生成故障含所有可观测位置；核查 UU 占比；observe/detect 同一 block。 |
| 9 | 工具本身可信吗？ | 出示功能安全认证/qualification kit + CoU/AoU 符合性检查表。 |
| 10 | 结果可复现吗？ | 保留脚本、SFF、种子、simv、报告、版本号；`save_session` 支持复跑。 |

---

## 12. 术语对照表

| 英文 | 中文 | 说明 |
|------|------|------|
| Fault Model | 故障模型 | Stuck‑At/Transient（功能安全）、Transition/Bridge（制造） |
| Stuck‑At (sa0/sa1) | 固定型故障 | 值钉死 0/1，6 类位置 |
| Transient | 瞬态/软错误故障 | 发生一次即消失，建模 SEU，放 FF 输出与存储器 |
| Transition (R/F) | 时延/跳变故障 | slow‑to‑rise / slow‑to‑fall（制造） |
| Bridge (wire/wor/wand) | 桥接故障 | 两 net 短接（制造） |
| Fault Universe / List | 故障宇宙 / 清单 | 全部待注入故障 |
| Fault Collapsing / Prime | 故障塌缩 / 源故障 | 等价故障合并 |
| Good / Faulty Machine (GM/FM) | 参考机 / 故障机 | 并发比较的两类实例 |
| Strobe | 选通点 | 判定差异的时间与信号 |
| Observe vs Detect | 观测点 vs 诊断点 | 是否传播到输出 vs 是否被 SM 捕获 |
| Safety Mechanism (SM) | 安全机制 | ECC、lockstep、CRC、STL 等 |
| UCFD tasks | 用户控制故障检测任务 | `$fs_compare`/`observe`/`detect`/`strobe` 等 |
| DG / Detected | 已检测组 | DT、DD(Dropped Detected)、DE 等 |
| Potential (PT/PD) | 潜在检测 | X 不确定，覆盖率按 0.5 折算 |
| Untestable (UT/UU/UB/UR/UI) | 不可测组 | = 功能安全里的安全故障 |
| Fault Dropping / maxhard | 故障丢弃 | 检测达阈值后不再仿真（默认 1 次） |
| UDFS | 用户自定义故障状态 | 功能安全语义（如 DD=Dangerous Diagnosed） |
| Test / Fault Coverage | 测试/故障覆盖率 | 分母是否含 Untestable |
| Diagnostic Coverage (DC) | 诊断覆盖率 | SFF 自定义方程，喂 SPFM/LFM |
| SPFM / LFM / PMHF | 单点/潜伏故障度量、随机失效概率 | ISO 26262 硬件指标 |
| SFF | 标准故障格式 | 输入输出核心文件 |
| FCM / vc_fcc / FDB | 故障活动管理器 / 编译器 / 数据库 | 流程总控 / 生成清单 / 存结果 |
| CoU / AoU | 使用条件 / 使用假设 | 功能安全强制遵守 |

---

*本手册基于本地 RAG 检索的官方文档片段（VC Z01X User Guide 与 Manufacturing Assurance Guide, X‑2025.06）整理。
命令语法、状态码与覆盖率公式的权威定义，请以你所用 Z01X 版本的 User Guide 正文与附录 A/B 为准。*
