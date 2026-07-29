# VC FSM Tech Data 建模报告 —— 28nm 车规集成 ISP

> 面向 ISO 26262 FMEDA，基于 Synopsys VC Functional Safety Manager 的 Tech Data / IEC 62380 BFR 设置。
> 本报告整理了：Tech Data 多实例机制、永久 vs 瞬态失效率模型、IEC 62380 Table 16 (λ1/λ2) 器件族选型、
> 28nm 瞬态 SER 数据、以及 IEC 62380 Parameters + Mission Profile 的完整参数设置。
>
> **器件对象**：28nm CMOS **集成 ISP**（自研标准单元 ASIC；图像数据通路 + 片上 SRAM/ROM + PLL/PHY 模拟）。
> **免责**：文中所有失效率数值为**行业典型量级估计**，正式 FMEDA 须用 foundry / JESD89 实测数据与整车
> mission profile 替换，并按 ISO 26262-11 §4.6.1.7 登记为 Analysis Assumption (AoU)。

---

## 目录
1. [Tech Data 是什么 & 多实例机制](#1-tech-data-是什么--多实例机制)
2. [永久 vs 瞬态：两套独立模型](#2-永久-vs-瞬态两套独立模型)
3. [IEC 62380 Table 16：λ1/λ2 器件族选型](#3-iec-62380-table-16λ1λ2-器件族选型)
4. [28nm 瞬态 SER 数据](#4-28nm-瞬态-ser-数据)
5. [IEC 62380 Parameters 设置](#5-iec-62380-parameters-设置)
6. [Mission Profile 设置](#6-mission-profile-设置)
7. [可直接运行的 tcl](#7-可直接运行的-tcl)
8. [必须用真值替换的项 & AoU 清单](#8-必须用真值替换的项--aou-清单)

---

## 1. Tech Data 是什么 & 多实例机制

Tech Data = VC FSM 的**失效率库（FIT rate library）**，把"工艺/资源类型"映射到"每单位失效率"，是 FMEDA 定量计算的 λ 源头。

核心列：

| 列 | 含义 |
|---|---|
| `Type` | 资源类型（Flops / RAM_Bits / Digital_Area…） |
| `PermanentFailureRate` | 永久故障 FIT（每单位） |
| `TransientFailureRate` | 瞬态故障 FIT（每单位） |
| `Unit` | 单位（FIT per Flop / per µm²…） |
| **`MappedDesignKey`** | 关联到 Design Data 的哪一列（把 FIT 和"数量"挂钩） |

**λ 计算**：`元件 λ = Σ (Design Data 数量 × Tech Data 对应 FIT)`，分永久/瞬态。
即 Tech Data 给"单价"，Design Data 给"数量"，相乘得元件失效率 → FMEDA → SPFM/LFM/PMHF。

**多实例（instance）**：Tech Data 可建多套，每套对应一种工况/工艺/标准：
```tcl
create_instance -table tech -instance_list "iec62380_isp_worldwide"
sync_bfr_setting -instance <bfr名> -tech_data_instance "iec62380_isp_worldwide"
```
典型用途：不同温度剖面（85°C vs 125°C）、不同工艺节点、IEC 62380 与 SN29500 各一套，用于对比或多配置分析。

---

## 2. 永久 vs 瞬态：两套独立模型

| | 永久 (Permanent) | 瞬态 (Transient / 软错误) |
|---|---|---|
| 物理机理 | 磨损、CTE 失配、热循环疲劳 | α/β/中子翻转存储节点（SEU/SET） |
| 算法来源 | **IEC 62380 / SN29500 温度剖面模型** | **JESD89 软错误表征**（中子通量/海拔/电压） |
| 受温度剖面驱动 | **是**（结温 + 热循环） | 基本无关（弱温度依赖） |
| 标准出处 | ISO 26262-11 §4.6.1；IEC 62380 | ISO 26262-11 §4.6.1.8 / §5.2.2.2 |

> 结论：**温度剖面只驱动永久失效率**；瞬态失效率是另一套机制（见 §4）。

**瞬态在 VC FSM 里的两种算法**：
```
TransientIsCustomized == true  → 瞬态FIT/unit = TransientFitPerUnit（直接填 SER 值）
TransientIsCustomized == false → 瞬态FIT/unit = TransientProportionOfPermanent × 永久FIT/unit
```
⚠️ **对 flop / SRAM，永久率≈0 → 比例法会算出 0**，必须用机制 A（Customized 直接填 SER）。

瞬态往下：FMEA 中 `FailureClass2=Transient` 的失效模式 → DcKrf 取 sm_type 的 **TranKrf** 列 → 汇入 **SPFM**；
**不进 LFM**（软错误自消失、不潜伏，故 sm_type 无 TranKmpf 列）。

---

## 3. IEC 62380 Table 16：λ1/λ2 器件族选型

**Table 16 = "Values of λ1 and λ2 for integrated circuits families"**（IEC TR 62380:2004，印刷页 33）。
- **λ1**：与 die 相关的基础失效率（FIT），随晶体管数 N 起作用
- **λ2**：与工艺成熟度（technology mastering）相关的失效率（FIT）
- **N**：代表性晶体管数计法（per gate / per bit / per programmable point）

VC FSM 里对应 `set_bfr_setting -tech_factor -field TechTable -value "family::type"`，即在 Table 16 里选行取 λ1/λ2。

### 关键选型原则
**按"设计类型"选，不是按"功能"选**：ISP 虽做图像 DSP 运算，但它是**自研标准单元 ASIC**，
应选 `MOS : Asic circuits`，而非 `MOS : Standard circuits::Digital, Micros, DSP`（后者指采购的标准成品芯片）。

### 本 ISP 的锁定映射（28nm CMOS）

| # | Tech Data Type | TechUnitType | 选定 Table 16 `type` | λ1 | λ2 |
|---|---|---|---|---|---|
| 1 | Flops | Digital::Flops | `MOS : Asic circuits::Standard Cell, Full Custom` | 1.2e‑5 | 10 |
| 2 | Latches | Digital::Latch | `MOS : Asic circuits::Standard Cell, Full Custom` | 1.2e‑5 | 10 |
| 5 | Digital_Area | Digital::Area | `MOS : Asic circuits::Standard Cell, Full Custom` | 1.2e‑5 | 10 |
| 3 | RAM_Bits | Memory::Bits | `MOS : Standard circuits::Static RAM - Low consumption (CMOS)` | 1.7e‑7 | 8.8 |
| 6 | Memory_Area | Transistor | 同 RAM_Bits（若启用） | 1.7e‑7 | 8.8 |
| 8 | RAM_Equiv_Xtors | Transistor | 同 RAM_Bits（若启用） | 1.7e‑7 | 8.8 |
| 4 | ROM_Bits | Memory::Bits | `MOS : Standard circuits::Read only memory` | 1.7e‑7 | 8.8 |
| 9 | ROM_Equiv_Xtors | Transistor | 同 ROM_Bits（若启用） | 1.7e‑7 | 8.8 |
| 7 | Analog_Area | Transistor | `MOS : Standard circuits::Linear circuits`（PLL/PHY 为主，纯模拟） | 0.01 | 4.2 |
| 10 | Package | Package | **不在 Table 16**（λ3 / Table 17a-b + 封装参数） | — | — |

**注意事项**：
- 存储器 `_Bits` / `_Equiv_Xtors` / `Memory_Area` **三选一启用**，否则永久率翻倍。
- 逻辑 `Digital_Area` 与 `Flops+Latches` 别把时序单元算两遍。
- SRAM 若为高速缓存，改选 `Static RAM - FIFO ("mixed MOS")`（N=4/bit），λ1/λ2 相同。
- Analog 若含 ADC/DAC 混合信号，改 `MOS : Standard circuits::Digital / linear circuits (…RAMDAC…)`（λ1 2.7e‑4 / λ2 20）；本 ISP 以 PLL/PHY 纯模拟为主，故选 Linear circuits。

---

## 4. 28nm 瞬态 SER 数据

**口径**：28nm bulk planar CMOS，未加固；地面海平面（纽约参考中子通量 ≈14 n/(cm²·h)，JESD89）；标称 Vdd；25°C。

| VC FSM Type | 单位 | 代表值 (FIT/unit) | 典型范围 | 依据 |
|---|---|---|---|---|
| Flops | FIT/flop | 3.0e‑4 | 1e‑4 – 1e‑3 | DFF SEU |
| Latches | FIT/latch | 2.0e‑4 | 1e‑4 – 5e‑4 | latch SEU |
| RAM_Bits | FIT/bit | 1.0e‑4 | 5e‑5 – 2e‑4 | SRAM ≈100 FIT/Mbit |
| ROM_Bits | FIT/bit | 0 (≈1e‑8) | ~0 | mask ROM 免疫 |
| Flash_Bits | FIT/bit | 1.0e‑6 | 1e‑7 – 1e‑5 | NVM 存储电荷 |
| Digital_Area (组合 SET) | FIT/µm² | 1.0e‑9 | 5e‑10 – 5e‑9 | 组合 SET，建议按 flop 比例 0.1~0.3× 建模或并入 flop |
| Analog_Area | FIT/µm² | 1.0e‑8 | 设计相关 | 模拟 SET，需按具体电路评估 |

**设置（机制 A，推荐）**：
```tcl
set_bfr_setting -instance $inst -tech_factor -tech_name Flops -field TechUnitType -value Customized
set_bfr_setting -instance $inst -tech_factor -tech_name Flops -field TransientIsCustomized -value true
set_bfr_setting -instance $inst -tech_factor -tech_name Flops -field TransientFitPerUnit   -value 3.0e-4
# RAM_Bits→1.0e-4, Latches→2.0e-4, Flash_Bits→1.0e-6 同理
```

---

## 5. IEC 62380 Parameters 设置

对应 GUI `Parameters...` 按钮。

### 结温模型（理解参数的核心）
```
ΔTj  = Rth(ja) × P                                  自升温
Tj(i)= tac(i) + ΔTj                                 第 i 相位结温 = 板级环境温 + 自升温
πt(i)= exp[ 3480 · (1/328 − 1/(273+Tj(i))) ]        Arrhenius，参考 55°C(328K)，Ea≈0.3eV
```
- Rth(ja) 由封装类型 + 引脚数 + 气流决定（自然对流 K=1.4）。
- πt 对高温极敏感（原文算例 Tj=118°C 时 πt=5.5）。
- 最敏感的三个输入：`DissipatedPower`、封装热阻、mission 高温相位占比。

### 参数表（28nm ISP 示例，`<>` 需替换实测值）

| 字段 | 含义 / 作用 | 建议值 |
|---|---|---|
| `PinCount` | 引脚数 S → Rth、λ3 | `<484>` |
| `DissipatedPower` | 功耗 P (W) → ΔTj（最敏感） | `<2.0>` |
| `TemperatureFactorType` | πt 曲线（活化能） | `MOS BiCMOS (low voltage)` |
| `JunctionResistanceType` | 封装类型 → Rth(ja) | `<你的封装, 如 PBGA/FCBGA>` |
| `JunctionResistanceMethod` | Rja / Rjc | `Rja`（有整体热阻）或 `Rjc` |
| `AirFlowType` | 气流 → K | `Natural convection` |
| `ComponentType` | 封装材料 | `Epoxy (Plastic package)` |
| `SubstrateType` | PCB 基材 → πα | `Epoxy Glass (FR4, G-10)` |
| `Lambda3Type` | 封装 λ3 连接类型（Table 17） | 按封装（BGA 面阵列类型） |
| `Lambda3Pitch` / `Lambda3Width` | 焊球间距/尺寸 (mm) | `<按封装>` |
| `ManufacturingYear` | a=(年−1998)，老化/成熟度 | `<2024>` |
| `QualifiedYear` | 认证年 | `<2023>` |
| `OverstressType` | λ_EOS，接口/非接口 | 核心逻辑 `Non Interfaces::All electrical environment`；I/O 密集块按接口 |

> ⚠️ `JunctionResistanceType` 必须选真实封装，DIL/PQFP 与 FCBGA 的 Rth 差异巨大。

---

## 6. Mission Profile 设置

对应 GUI `Mission Profile...` 按钮。分**工作相位**（→ λ_die）和**循环相位**（→ λ_package）。

### 工作相位 WorkingPhase（车规 ISP 示例）
一组"环境温度 + 年时间占比"（和 = Won；剩余为存储 Woff）。

| 相位 | 板级环境温 tac (°C) | 年占比 |
|---|---|---|
| 常温运行 | 40 | 0.30 |
| 典型高温 | 85 | 0.45 |
| 高温 | 105 | 0.15 |
| 极限高温 | 125 | 0.05 |
| 存储 (Woff) | — | 0.05 |

### 循环相位 CyclingPhase
三个量：年循环次数 n、循环周期 θ(h)、温度幅度 ΔT(°C)。
车规典型：ΔT 大、每天约 2 次开关机。示例 `{730, 12, 80}`。

> ⚠️ 三个数的确切顺序请在 GUI `Mission Profile...` 对话框核对列标题（版本间排列可能不同；demo 为 `{670 1340 30}`）。

---

## 7. 可直接运行的 tcl

```tcl
set inst iec_isp_28nm
create_bfr_setting -instance $inst

# --- Parameters ---
set_bfr_setting -instance $inst -parameter -field DissipatedPower          -value 2.0
set_bfr_setting -instance $inst -parameter -field PinCount                 -value 484
set_bfr_setting -instance $inst -parameter -field TemperatureFactorType    -value "MOS BiCMOS (low voltage)"
set_bfr_setting -instance $inst -parameter -field AirFlowType              -value "Natural convection"
set_bfr_setting -instance $inst -parameter -field JunctionResistanceType   -value "<你的封装, e.g. PBGA>"
set_bfr_setting -instance $inst -parameter -field JunctionResistanceMethod -value "Rja"
set_bfr_setting -instance $inst -parameter -field ManufacturingYear        -value 2024
set_bfr_setting -instance $inst -parameter -field OverstressType           -value "Non Interfaces::All electrical environment"

# --- Mission Profile ---
set_bfr_setting -instance $inst -profile -field WorkingPhaseAnnualRatioMethod -value "By Annual Ratio"
set_bfr_setting -instance $inst -profile -field WorkingPhaseList -value [list \
   "Temperature" [list 40 85 105 125] "AnnualRatio" [list 0.30 0.45 0.15 0.05]]
set_bfr_setting -instance $inst -profile -field CyclingPhaseType -value "Automotive"
set_bfr_setting -instance $inst -profile -field CyclingPhaseClimateType -value "World-wide"
set_bfr_setting -instance $inst -profile -field CyclingPhaseList -value [list "Cycle" [list 730 12 80]]

# --- TechTable（永久 λ1/λ2 选型）---
set_bfr_setting -instance $inst -tech_factor -tech_name Flops        -field TechTable -value "MOS : Asic circuits::Standard Cell, Full Custom"
set_bfr_setting -instance $inst -tech_factor -tech_name Latches      -field TechTable -value "MOS : Asic circuits::Standard Cell, Full Custom"
set_bfr_setting -instance $inst -tech_factor -tech_name Digital_Area -field TechTable -value "MOS : Asic circuits::Standard Cell, Full Custom"
set_bfr_setting -instance $inst -tech_factor -tech_name RAM_Bits     -field TechTable -value "MOS : Standard circuits::Static Read Access Memory - Low consumption; (CMOS)"
set_bfr_setting -instance $inst -tech_factor -tech_name ROM_Bits     -field TechTable -value "MOS : Standard circuits::Read only memory"
set_bfr_setting -instance $inst -tech_factor -tech_name Analog_Area  -field TechTable -value "MOS : Standard circuits::Linear circuits"

# --- 瞬态 SER（机制 A，28nm）---
foreach {t v} {Flops 3.0e-4 Latches 2.0e-4 RAM_Bits 1.0e-4 Flash_Bits 1.0e-6} {
  set_bfr_setting -instance $inst -tech_factor -tech_name $t -field TransientIsCustomized -value true
  set_bfr_setting -instance $inst -tech_factor -tech_name $t -field TransientFitPerUnit   -value $v
}

# --- 同步到 Tech Data + 出结果 ---
create_instance -table tech -instance_list $inst
sync_bfr_setting -instance $inst -tech_data_instance $inst
```
设完在 GUI 点 `Update Counts...` 拉 N，再看 `λ Details...` 核对 λ_die / λ_package / λ_overstress。

---

## 8. 必须用真值替换的项 & AoU 清单

**四个决定结果量级、必须用实测替换的输入**：
1. `DissipatedPower`（功耗 P）
2. `JunctionResistanceType`（真实封装 → Rth）
3. Mission Profile 的**高温相位年占比**
4. 循环相位 `ΔT / n`

**登记为 Analysis Assumption (AoU)，拿到数据后更新**（ISO 26262-11 §4.6.1.7）：
- 全部瞬态 SER 值 → 用 foundry SER report / JESD89 实测 / 供应商 datasheet 替换；偏低会使 SPFM 虚高（危险乐观），不确定时取范围上限保守。
- Mission profile → 用整车实际 mission profile（LV124 等）替换。
- λ1/λ2 器件族选型 → 与安全团队确认 Standard Cell vs 其它族。

**版权提醒**：IEC TR 62380 / ISO 26262 为 IEC/ISO 授权文档，文中转录的 λ1/λ2 数值仅供内部学习建模，请勿对外再分发原始表页。

---

## 附录 A：四张 FMEA/FMEDA 表的 PSM/LSM 与 DcKrf/DcKmpf 填法速查

### A.1 核心概念

- **PSM（Primary Safety Mechanism，主安全机制）** → 覆盖**单点/残余故障** → 提供 **DcKrf** → 驱动 **SPFM**。
- **LSM（Latent Safety Mechanism，潜伏安全机制）** → 覆盖**潜伏/多点故障** → 提供 **DcKmpf** → 驱动 **LFM**。
- **FMEA = 定性**（分配哪个 PSM/LSM，填 SM 的 ID）；**FMEDA = 定量**（分配 SM Type，得 DcKrf/DcKmpf 数值）。
- **Main = 分析功能失效**；**SM = 分析安全机制自身失效**（后者本质是潜伏故障）。
- DC 定性分级（ISO 26262‑5 p.52）：**Low/Medium/High = 60% / 90% / 99%**，用于为定量 DcKrf/DcKmpf 提供依据。

### A.2 填法速查表

| 表 | PSM 侧（Current PSM / PSMType / DcKrf） | LSM 侧（Current LSM / LSMType / DcKmpf） |
|---|---|---|
| **Main FMEA**（功能，定性）| **要填**：功能单点故障由哪个 PSM 覆盖 | 视情况：功能潜伏故障由哪个 LSM 覆盖 |
| **Main FMEDA**（功能，定量）| **要填**：PSMType → DcKrf（→ SPFM）| 视情况：LSMType → DcKmpf（→ LFM，功能侧潜伏）|
| **SM FMEA**（机制自身，定性）| **一般留空**（例外：SM 失效直接构成单点故障）| **要填**：SM 自身潜伏失效由哪个 LSM 兜底 |
| **SM FMEDA**（机制自身，定量）| **一般留空 = No SM assigned, DcKrf=0** | **要填**：LSMType → DcKmpf（→ LFM，机制侧潜伏）|

> demo 实测印证：Main FMEDA `DcKrf=0.9 / DcKmpf=0(No SM assigned)`；SM FMEDA `DcKrf=0(No SM assigned) / DcKmpf=0.99`。

### A.3 两个"LSM"的关系（Main FMEDA vs SM FMEDA）

一个多点/潜伏故障 = **功能故障 + 安全机制故障** 两个独立故障叠加。潜伏检测分两侧记账：

```
双点故障 = 功能坏  +  保护它的 SM 坏
            │            │
   Main FMEDA 的 LSM   SM FMEDA 的 LSM
   (功能侧潜伏)         (机制自身潜伏)
            └──── 都汇入同一个 LFM ────┘
```

- **同一份 LSM 目录**：两张表的 LSM 都从 PSM/LSM 清单里选（demo 为 `HOST_LSM_1/2`）。
- **汇入同一个 LFM**：整芯片 LFM = Main FMEDA（功能潜伏）+ SM FMEDA（机制潜伏）求和。
- **常是同一个物理机制**：一个上电/周期性自检（MBIST/LBIST/SM self‑test）经常同时查"SM 还活着没"（SM FMEDA）和"功能里有没有被掩盖的潜伏缺陷"（Main FMEDA）→ **同一个 LSM ID 可同时出现在两张表**。
- 但**分开记账、作用于不同失效率来源**：Main 侧 DcKmpf × 功能多点故障池；SM 侧 DcKmpf × 安全机制自身失效率。

### A.4 关键判定口诀

- **SM 自身失效 = 潜伏故障** → 只走 **LSM/DcKmpf/LFM**，不走 PSM/DcKrf/SPFM（故 SM FMEA/FMEDA 的 PSM 侧留空）。
- **瞬态故障** → 只走 DcKrf(TranKrf)/SPFM，**不进 LFM**（sm_type 无 TranKmpf）。
- **Estimated DC（定性 High/Med/Low）** 不直接进公式，但应填作为定量 DC 的依据。

### A.5 ISP 具体示例（line buffer SRAM + Parity 校验器）

| 分析对象 | 表 | PSM | LSM |
|---|---|---|---|
| SRAM 单点故障 | Main FMEA/FMEDA | Parity 校验器（DcKrf → SPFM）| — |
| SRAM 潜伏故障（被 Parity 掩盖的 stuck bit）| Main FMEDA | — | MBIST/memory scrub（DcKmpf → LFM）|
| Parity 校验器自身静默失效 | SM FMEA/FMEDA | 留空 | 校验器自检 / 同一 MBIST（DcKmpf → LFM）|

---

*本报告依据 VC FSM examples 实际 API、IEC TR 62380:2004、ISO 26262‑5/11:2018 整理，供 28nm 车规 ISP 的 FMEDA 建模参考。*
