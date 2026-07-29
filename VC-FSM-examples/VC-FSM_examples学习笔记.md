# Synopsys VC Functional Safety Manager (VC FSM) —— Examples 学习笔记

> 本文基于 `vc-fsm/examples/` 下的全部示例整理，覆盖：工具架构、各示例用途与文件含义、
> IPData 中 FMEA/FMEDA/PSM/LSM 表的语义，以及 `DcKrf/DcKmpf` 如何经 `sm_type.csv`
> 换算成 ISO 26262 的 SPFM/LFM 指标。

---

## 目录

1. [VC FSM 是什么](#一vc-fsm-是什么)
2. [示例分两大类](#二示例分两大类)
3. [A 类：完整 Client/Server 流程 demo](#三a-类完整-clientserver-流程-demo)
4. [B 类：独立工具脚本](#四b-类独立工具脚本)
5. [典型跑法](#五典型跑法以-demo-为例)
6. [全局数据流关系](#六全局数据流关系)
7. [IPData 核心表语义：FMEA / FMEDA / PSM / LSM](#七ipdata-核心表语义fmea--fmeda--psm--lsm)
8. [DcKrf/DcKmpf 经 sm_type 换算成 SPFM/LFM](#八dckrfdckmpf-经-sm_type-换算成-spfmlfm)

---

## 一、VC FSM 是什么

Synopsys 的**功能安全 FMEDA 数据库管理工具**，服务 ISO 26262（车规）等安全流程。
核心是一套 **Client/Server 架构**：

- **`vc_fsm_server`**：后台服务，管理一个 FMEDA 数据库（`fmedaDB/`），处理用户/权限/版本。
- **`vc_fsm_client`**：GUI 或 `--batch` 批处理客户端，通过 **Tcl 命令**操作数据库。
- 数据模型是一组**表**：Design Data（设计层次/面积/触发器数）、Tech Data（工艺失效率 FIT）、
  SM Type（安全机制诊断覆盖率）、Synthetic Hierarchy（综合层次）、Main/SM FMEA、
  Main/SM FMEDA、FI Effort（故障注入活动）等。

它算的核心指标是 ISO 26262 的 **SPFM / LFM / PMHF**，输入来自各表的
`Fsafe`、`DC(Krf)`、`DC(Kmpf)`、`FIT` 等。

---

## 二、示例分两大类

| 类别 | 示例 | 说明 |
|---|---|---|
| **A. 完整 Client/Server 流程 demo** | `demo/`、`HL-fRMEM/`、`apps/FaultInjection…`、`apps/DesignDataBatchImport/` | 起 server → 导入数据 → 跑客户端。有整套启停脚本 |
| **B. 独立 Tcl / 工具脚本** | `IEC62380/`、`SN29500/`、`ruleChecker/`、`netlist_extract/`、`quickSynthesis/`、`Spyglass_FA/`、`fdb/` | 单个功能的脚本模板，配合主流程用 |

---

## 三、A 类：完整 Client/Server 流程 demo

### `demo/` —— 最基础的入门例子（门级 FMEDA）

一个自动驾驶芯片 `SelfDrvChip` 的 FMEDA。**每个 demo 目录都是同一套脚手架**：

| 文件 | 作用 |
|---|---|
| `ServerSettings` | 设 `VC_FSM_HOST_PORT`（9990）/`VC_FSM_DB_PORT`（9980）两个空闲 TCP 端口、host 名、管理员账号 `pm`/密码 |
| `cfg_first.ini`(`.org`) | Server 配置：定义 project、DB 路径、用户及其可访问 IP 权限。`.org` 是原始模板，`cleanUp` 会用它覆盖回去 |
| `startServer`/`stopServer` | 起停 server（`vc_fsm_server --conf=… --start/--stop`） |
| `startClient`/`startClientAsAdmin` | 普通用户 / 管理员登录 GUI |
| `importDemoData` | `vc_fsm_client --batch --admin --do import_data.tcl` 批量导入 |
| `cleanUp` | 删 `fmedaDB`、log、xml、xlsx，重置 ini |

**数据组织方式**（理解全工具的关键）——`import_data.tcl` 是导入主脚本：

1. `source customized_design.tcl` —— 定义 Design Data 的**自定义列**（Area/Flops/Latches/RAM Bits…）
   以及 Tech 表到设计列的**映射**（`MappedDesignKey`）。
2. `import_data -table tech -file techdata.csv` —— 导入**工艺失效率**（每类资源的 permanent/transient FIT）。
3. `-table sm_type` —— 导入**安全机制类型**及其 Krf/Kmpf 诊断覆盖率（如 ECC、Parity）。
4. `-table synthetic_hierarchy` —— 导入**综合层次**（component/part/subpart 树）。
5. 设 `ScenarioList`/`WorkloadList` —— 故障注入的场景和 workload 类型。
6. 按 IP 导入 Design Data 和 FMEA/FMEDA（`IPData/HOST/` 下一堆 CSV）。

`IPData/HOST/*.csv` 是单个 IP 的全部安全数据：`HOST_Design_Hier.csv`（设计层次）、
`HOST_Element.csv`（元件）、`HOST_PSM/LSM.csv`（主/潜伏安全机制）、
`HOST_Main_FMEDA.csv`（每个失效模式的 Fsafe/DC/Fpvsg）、`*_Change_History.csv`（版本历史）。

`HOST_Design_Data_Tag.yml` —— 给这批设计数据打的**版本 tag**（netlist 来源、revision、上传人），做数据溯源。

> README 第 7 步特意说：更新 metric 时会出现 `Cannot find design instance 'ALU0'…Use base design data`
> 的 warning——这是演示"设计实例不存在时回退到 base data"的正常行为，不是 bug。

### `HL-fRMEM/` —— 高层概念 FMEDA（High Level Concept FMEDA）

比 `demo` 更抽象一层：在没有门级网表、只有架构概念时做 FMEDA。

- 综合层次的叶子节点是 **Top Level Failure Mode + Analysis Assumption**，可配置假设
  （实例数、Safeness、是否安全相关、SM 分配、HW 元件类别）。
- 导入简化成**一条命令**：`::fsm_utility::import_project_data -directory ProjectData -log_directory Log`，
  直接吃 `ProjectData/` 下整套标准命名的 CSV（`Safety_Goal.csv`、`Requirement.csv`、
  `Failure_Mode_Effect.csv`、`Configuration.csv`、各种 `Syn_Hier_*.csv` 等 = **完整项目数据库的导出/导入格式**）。
- GUI 里 `Project → High Level Concept FMEDA` 打开对应窗口。

### `apps/FaultInjectionAndResultsBackAnnotation/` —— 故障注入闭环（重点应用）

把 VC FSM 和外部故障仿真（Z01X 等）打通的**双向数据流**，靠 `$VC_FSM_APPS_PATH` 下两个封装脚本：

1. **查询要注入的 FMEDA 数据** `queryFMEDADataForFI`
   - 输入 `input_to_snps_query_fmeda.yml`：列出要查哪些 IP 及 change（`current`/`latest`/具体编号）。
   - 输出 `output_from_snps_fmeda_list.yml`，自研脚本据此生成故障仿真配置。
2. **把仿真测出的参数回标（back-annotate）** `saveFIResults`
   - 用故障仿真测出 `Fsafe`、`DC(Krf)`、`DC(Kmpf)` → 存进 CSV。
   - `--dry_run` 先校验合法性，再正式写回 FMEDA 并生成 change history。

- `set_fi_result_columns.tcl` 定义 FI Results 表的列（DUT 名、注入实例、ZOIX 版本、参数名/值、结果…）。
- `testFIRBAL` 是端到端自测脚本（error case → dry run → 正式保存）。

### `apps/DesignDataBatchImport/` —— 设计数据批量导入的 YAML 格式

演示 `dd_tag` YAML 怎么把**多个设计数据文件映射到 IP 实例**：

- `type: superset`（IP 的全集基准）vs `type: instance`（具体实例，可各自用不同 CSV）。
- `test1/2/3.yaml` 是正确用法；`test_error1/2.yaml` 是**故意的错误用法**
  （`invalid_type`、不存在的文件、不存在的 IP），用来演示工具的报错校验。
- `DD1.csv`/`DD2.csv` 是门级提取出的设计统计（HierScope、Area、Flops、各种 Accumulation 累计列）。

---

## 四、B 类：独立工具脚本

### 可靠性预测：`IEC62380/` 和 `SN29500/`（算基础失效率 BFR）

两个工业标准的**基础失效率（Base Failure Rate）计算器**，纯 Tcl，套路一致：

1. `create_bfr_setting` 建一个 setting 实例（SN29500 加 `-sn29500`）。
2. `set_bfr_setting -parameter` 设参数（引脚数、功耗、结温、任务剖面 mission profile、工作/循环相位）。
3. `set_bfr_setting -tech_factor` 逐个工艺（Digital_Area/Flops/RAM/ROM/Flash/Analog/Package/Overstress）设晶体管数和工艺表。
4. `get_bfr_setting -intermediate_value` 读结果（λDie、λPackage，SN29500 还有 πD/πT/πU/πW/πF 各因子）。
5. **`sync_bfr_setting -tech_data_instance` 把算出的 FIT 同步回 Tech Data 表** —— 接入主流程的接口。

区别：IEC62380 侧重任务剖面（温度循环 + 年占比），SN29500 侧重 π 因子模型。

### `ruleChecker/` —— 自定义数据校验规则

- `example_rules.tcl`：`register_rule` 注册规则（tag、severity、作用域 IP、作用表、回调），
  回调里遍历行、用正则查 Failure Mode ID 是否符合命名规范（如 `IPName_FM_123`），不符就 `add_check_result` 报错。
- `gen_check_report.tcl` / `gen_check_report_in_yaml.tcl`：把 `get_check_result_list` 的结果导出成文本 / YAML 报告。

### `netlist_extract/genDesignMapFileFromStdCellLib/` —— 从标准单元库提取面积/触发器

- 用 **Design Compiler**（`dc_shell`）跑 `test_extract.tcl`（列出 `.db` 库）→ `extract_cell_lib.tcl`
  遍历每个 cell，读 area、是否时序、multibit 位宽 → 输出 `cell_lib.csv`
  （`Type,Name,Parameters,Digital_Area,Flops,RAM_Bits`）。
- 这个 CSV 就是后面 `netlist_extract` 工具**把门级网表映射成设计统计**用的"字典"。`make clean run` 一把跑。

### `quickSynthesis/dc_run_template.tcl` —— 快速综合模板

DC 综合模板，`__USER_XXX__` 是占位符（VC FSM 替换成实际库/顶层/源文件），综合完输出 `.vg`/`.ddc` 和 area 报告。
目的：从 RTL 快速拿到面积/触发器数喂给 FMEDA。

### `Spyglass_FA/` —— 对接 SpyGlass Fault Analysis

- `template.prj`：SpyGlass 工程模板（`__USER_XXX__` 占位符）。定义 `vc_fsm_goal`，按开关 dump：
  设计单元清单、每个失效模式的 COI（Cone of Influence）、TMR 寄存器选择的 SSF、SPFM 目标百分比。
- `cells_dmap.csv`：RTL 级 cell 到面积/触发器的映射表（`RTL_AND`、`RTL_NOT`…）。

### `fdb/template.sff` —— 故障状态格式（Status Fault Format）

Z01X / VCF / ZeBu 故障仿真的**故障状态定义模板**：

- 定义所有故障状态、状态分组（AF/AU/DN/DD/FN…）、**促升规则 PromotionTable**（多工具结果合并优先级）。
- 底部 **Coverage 公式**：直接给出 `Fsafe`、`DC(Krf)`、`DC(Kmpf)`、`Max(DC)` 的分子分母定义
  —— 故障仿真结果如何换算成 ISO 26262 指标的规则。

---

## 五、典型跑法（以 demo 为例）

```bash
# 1. 环境
setenv VC_FSM_HOME <安装路径>；set path=($VC_FSM_HOME/bin $path)
setenv SNPSLMD_LICENSE_FILE <license>
export HOST=$(hostname)          # ServerSettings 需要 $HOST

# 2. 进 demo/，重置 + 改端口/账号
./cleanUp
vi ServerSettings               # 改成两个空闲端口
vi cfg_first.ini                # 改用户名/密码/IP 权限

# 3. 起 server → 另开终端导数据 → 起客户端
./startServer
./importDemoData
./startClientAsAdmin            # 或 ./startClient（普通用户）

# 4. 收尾
./stopServer
./cleanUp
```

FaultInjection app 多两步环境变量（`$VC_FSM_APPS_PATH`），然后 `queryFMEDADataForFI` / `saveFIResults`。

---

## 六、全局数据流关系

```
RTL/网表 ──DC(quickSynthesis/netlist_extract)──► Design Data(面积/Flops)
标准手册 ──IEC62380/SN29500──► Tech Data(FIT)          ┐
SM 定义   ──sm_type.csv──► SM Type(诊断覆盖率)          ├─► VC FSM DB ──► SPFM/LFM/PMHF
故障仿真  ──Spyglass_FA/fdb(SFF)──► DC(Krf/Kmpf)/Fsafe ┘         ▲
                                    ▲                            │
              FaultInjection app(query→仿真→saveFIResults 回标)──┘
              ruleChecker: 全程数据合规校验
```

---

## 七、IPData 核心表语义：FMEA / FMEDA / PSM / LSM

这几张表是一套标准 **ISO 26262 FMEDA** 的分工。核心是把"**功能**"和"**保护功能的安全机制**"分开，
再把"**定性分析(FMEA)**"和"**定量算指标(FMEDA)**"分开。

### 两条正交的划分线

```
                定性 (FMEA)            定量 (FMEDA)
              失效模式/原因/后果        失效率分配/DC/指标
            ┌─────────────────┬─────────────────────┐
 功能本身    │   Main_FMEA     │    Main_FMEDA        │
            ├─────────────────┼─────────────────────┤
 安全机制    │   SM_FMEA       │    SM_FMEDA          │
            └─────────────────┴─────────────────────┘

 安全机制清单：PSM(主) + LSM(潜伏)
```

### PSM 和 LSM —— 安全机制清单（两类）

安全机制（Safety Mechanism, SM）= 设计里用来"抓故障"的保护逻辑（ECC、Parity、监控器…）。
按它**抓哪种故障**分两类：

- **`PSM` = Primary Safety Mechanism（主/一次安全机制）**
  - 抓**直接会违反安全目标的故障**，即 **Single-Point / Residual Fault（单点/残余故障）**。
  - 诊断覆盖率贡献给 **DC(Krf)** → 决定 **SPFM（单点故障度量）**。
  - 例：`HOST_PSM_1 "Host Safety 1"`，诊断方式 `Integrated hardware consistency monitoring`。
  - 表结构：挂在 **Element** 上（`Element` 列）。

- **`LSM` = Latent Safety Mechanism（潜伏安全机制）**
  - 抓**潜伏/多点故障（Multi-Point/Latent Fault）**——即"主安全机制自己坏了但没人发现"这类二阶故障。
  - 诊断覆盖率贡献给 **DC(Kmpf)** → 决定 **LFM（潜伏故障度量）**。
  - 例：`HOST_LSM_1 "Latent Safety 1"`，用来检测 PSM 失效。
  - 表结构：挂在**层次**上（`Hierarchy` 列）。

> 两张表列几乎一样：ID、SafetyMechanism、Diagnostic 类型、周期、执行时间、错误响应、
> `IsoDiagnostic`（ISO 诊断方法）、`EstimatedDC`（估计覆盖率）。区别在挂载对象与面对的故障阶数。

### Main_FMEA vs SM_FMEA —— 定性失效分析

- **`Main_FMEA` = 功能元件的失效模式分析**
  - 每行是**设计功能块的一个失效模式（FM）**，ID 形如 `HOST_FM_1`。
  - 关键列：
    - `PotentialFaults / PotentialErrors / PotentialEffect` —— 故障→错误→后果链
      （例："corrupted CPU command" → "Memory content corruption" → "Wrong execution"）。
    - `FailureClass2` —— Permanent / Transient（永久/瞬态）。
    - `SafetyRelated / SafeFailure / PVSG` —— 是否安全相关、是否安全失效、是否潜在违反安全目标。
    - **`CurrentPSMList / CurrentLSMList`** —— 该失效模式**由哪些 PSM / LSM 保护**。

- **`SM_FMEA` = 安全机制自身的失效模式分析**
  - 每行是**某个安全机制自己出问题的失效模式**，ID 形如 `HOST_SMF_1`。
  - `SafetyMechanism` 列指明是哪个 SM 坏了，如 `HOST_PSM_1::Host Safety 1`。
  - 失效模式典型是 `"Inadvertent activation"`（误触发），后果 `"Undetected safety-critical errors"`。
  - `CurrentLSMList` 指明**谁来兜底监控这个 SM 的失效**。
  - **这是"潜伏故障分析"的载体**——为什么需要 LSM 就体现在这里。

### Main_FMEDA vs SM_FMEDA —— 定量指标计算

FMEDA 表**逐行对应上面 FMEA 表的每个 FM**（靠 `FmeaID` / `SM_FmeaID` 关联），负责填数值算指标。
两张表列完全相同，关键列：

| 列 | 含义 |
|---|---|
| `FmeaID`/`SM_FmeaID` | 关联到对应 FMEA 行 |
| `Fsafe` | 安全失效占比（这部分故障本身无害） |
| `PSMType` + `DcKrf` | 分配的**主安全机制类型** → **对残余故障的诊断覆盖率**（→ SPFM） |
| `LSMType` + `DcKmpf` | 分配的**潜伏安全机制类型** → **对潜伏故障的诊断覆盖率**（→ LFM） |
| `Fpvsg` | 潜在违反安全目标的故障占比 |
| `Measured*` | 故障注入实测值（FI 回标写进来，对比估计值） |
| `FaultCampaignStatus` | 故障注入活动状态（`Not Run`…） |
| `Fper` | permanent 相关占比 |

- **Main_FMEDA**：算功能故障的 SPFM/LFM。例 `HOST_FM_2`：`Fsafe=0.5`、`DcKrf=0.9`（有 PSM 抓 90%）。
- **SM_FMEDA**：算安全机制自身故障的贡献。例 `HOST_SMF_1`：`DcKmpf=0.99`
  （SM 失效本质是潜伏故障，主要靠 DcKmpf/LSM 覆盖）。

`PSMType`/`LSMType` 的取值（如 `"Multiple SMs (Auto DC)"`、`"SM type not specified"`）来自
**SM_Type 表**（`sm_type.csv`），那张表定义每种 SM 类型的 `PermKrf/TranKrf/PermKmpf` 覆盖率数值。

### 串联示意（HOST_FM_3 为例）

```
Main_FMEA:  HOST_FM_3  "corrupted CPU write data"
            ├─ 由 PSM 保护: HOST_PSM_1  (抓单点/残余故障)
            └─ 由 LSM 保护: HOST_PSM_2 ...
                    │
   ┌────────────────┴─────────────────┐
   ▼                                   ▼
Main_FMEDA[FmeaID=HOST_FM_3]        PSM/LSM 清单表
   Fsafe / DcKrf(→SPFM) /             定义每个 SM 的诊断方法、DC
   DcKmpf(→LFM) / Fpvsg
                    │
   而 PSM 自己会不会坏？→ SM_FMEA(HOST_SMF_x "Inadvertent activation")
                       → 由 LSM(HOST_LSM_x) 兜底
                       → SM_FMEDA 算这部分潜伏故障的 DcKmpf
```

**一句话总结**：
- **PSM/LSM** = 安全机制两类清单（主抓单点故障→SPFM，潜伏抓多点故障→LFM）。
- **Main_\*** = 分析**功能**的失效；**SM_\*** = 分析**安全机制自身**的失效（潜伏故障来源）。
- **\*_FMEA** = 定性（故障链、分类、分配哪个 SM 保护）；
  **\*_FMEDA** = 定量（Fsafe/DC/Fpvsg，最终算 SPFM/LFM/PMHF）。

---

## 八、DcKrf/DcKmpf 经 sm_type 换算成 SPFM/LFM

核心分三步：**① sm_type 查出 DC → ② 每个 FM 把 λ(FIT) 分桶 → ③ 全芯片 FIT 加权汇总成 SPFM/LFM。**

### 1. sm_type.csv 的三列对应谁

```
"Name"                   PermKrf  TranKrf  PermKmpf
"Parity bus protection"  0.7      0.9      1
"ECC Mux 4"              0.2      0.9      0.73
```

三列 = **按"故障阶数 × 故障模型"切的诊断覆盖率 DC**：

| 列 | 给谁用 | 含义 |
|---|---|---|
| `PermKrf` | → **DcKrf**（永久故障时） | 主SM(PSM) 对**残余故障 residual**的覆盖率 |
| `TranKrf` | → **DcKrf**（瞬态故障时） | 同上，瞬态故障模型下的值 |
| `PermKmpf` | → **DcKmpf** | 潜伏SM(LSM) 对**潜伏/多点故障**的覆盖率 |

**关键点 1：为什么 Krf 有 Perm/Tran 两个，Kmpf 只有 Perm 一个？**
每个失效模式在 FMEA 里有 `FailureClass2 = Permanent / Transient`：
- Permanent FM → `DcKrf = PermKrf`；Transient FM → `DcKrf = TranKrf`。
- **瞬态故障会自己消失，不会"潜伏"**，故潜伏故障度量不考虑瞬态 → 没有 `TranKmpf` 列。

**关键点 2：DcKrf/DcKmpf 不一定来自查表。** 取决于该行选的 `PSMType`/`LSMType`：
- 选**具名 SM 类型**（如 `Parity bus protection`）→ VC FSM 自动填 sm_type 表对应值。
- 选 `Custom DC` / `SM type not specified` → `DcKrf`/`DcKmpf` **手填**
  （这解释了 `HOST_FM_2` 的 `PSMType="SM type not specified"`（表里 Krf=0）但 `DcKrf=0.9` 是手填的）。

### 2. 单个 FM：把 λ(FIT) 分成四个桶

每个失效模式先有总失效率 **λ**（Design Data 的 Flops/Area × Tech Data 的 FIT/unit）。
用 `Fsafe`、`Fpvsg`、`DcKrf`、`DcKmpf` 逐级切分：

```
λ (该 FM 总 FIT)
│
├─ 安全故障   λ_safe = λ · Fsafe                     → 不进危险桶
│
└─ 危险故障   λ_D = λ · (1−Fsafe) · Fpvsg            （会威胁安全目标的部分）
   │
   │   ← 主安全机制 PSM，覆盖率 DcKrf
   │
   ├─ 残余故障 RF   = λ_D · (1 − DcKrf)   ┐
   │   （没被主SM抓到）                    ├─►  进 SPFM 分子
   │   若该 FM 无 SM(DcKrf=0)，整块就是    │
   │   单点故障 SPF                        ┘
   │
   └─ 被主SM覆盖  λ_cov = λ_D · DcKrf
       │   ← 潜伏安全机制 LSM，覆盖率 DcKmpf
       │
       ├─ 潜伏故障 MPF_latent = λ_cov · (1 − DcKmpf) ─►  进 LFM 分子
       │   （主SM抓住了，但"主SM自己会不会坏"没人兜底）
       │
       └─ 已察觉/检测 MPF_detected = λ_cov · DcKmpf   → 安全
```

一句话：
- **DcKrf 决定"危险故障里有多少漏过主SM"**（漏的算 residual，进 SPFM）。
- **DcKmpf 决定"主SM覆盖的那部分里有多少变成潜伏"**（潜伏的进 LFM）。

### 3. 全芯片汇总：SPFM 和 LFM 公式

指标**按 FIT 加权、对所有安全相关 FM 求和**（在 Synthetic Hierarchy 逐层累加，用 syn_hier 表的
`PermanentFsr/TransientFsr/Fpvsg/Fsafe…` 分率列）：

**SPFM（单点故障度量）** —— 衡量"主SM 把危险单点故障挡掉多少"：

```
              Σ (λ_SPF + λ_RF)         Σ λ_D·(1 − DcKrf)
SPFM = 1  −  ───────────────────  = 1 − ──────────────────
                 Σ λ_危险                   Σ λ_D
```

**LFM（潜伏故障度量）** —— 衡量"LSM 把潜伏故障挡掉多少"，分母是**没被算成 SPF/RF 的故障**：

```
              Σ λ_MPF,latent          Σ λ_cov·(1 − DcKmpf)
LFM  = 1  −  ────────────────────  = 1 − ────────────────────
             Σ(λ_D − λ_SPF − λ_RF)         Σ λ_cov
```

**PMHF** ≈ `Σλ_RF/SPF + Σλ_MPF_latent` 的绝对 FIT 和，给硬件随机失效的绝对概率，用来判 ASIL 达标。

### 4. 数值小例子

某永久故障 FM：λ = 100 FIT，`Fsafe=0`，`Fpvsg=1`，用 **Parity bus protection**（PermKrf=0.7, PermKmpf=1）：

```
λ_D          = 100 · (1−0) · 1        = 100 FIT
DcKrf=0.7 →  RF        = 100·(1−0.7)  =  30 FIT   → 进 SPFM 分子
             λ_cov     = 100·0.7      =  70 FIT
DcKmpf=1  →  MPF_latent= 70·(1−1)     =   0 FIT   → 进 LFM 分子
```

若整芯片只有这一个危险 FM：
- SPFM = 1 − 30/100 = **70%**（正好等于 PermKrf，因为 Fsafe=0）
- LFM  = 1 − 0/70   = **100%**（Parity 的 Kmpf=1，潜伏全覆盖）

对比 **ECC Mux 4**（0.2 / 0.9 / 0.73）同样 100 FIT：
- RF = 100·0.8 = 80 → SPFM = 20%
- λ_cov = 20，MPF_latent = 20·(1−0.73)=5.4 → LFM = 1−5.4/20 = **73%**（就是 PermKmpf）

可见：**当 Fsafe=0、Fpvsg=1 时，单个 FM 的 SPFM 收敛到 DcKrf、LFM 收敛到 DcKmpf**——
验证了这两个 DC 就是 SPFM/LFM 的直接驱动量。真实芯片再叠加 Fsafe/Fpvsg 分流和多个 FM 的 FIT 加权即可。

---

*本文档由对 VC FSM examples 的逐文件分析整理而成，供学习参考。*
