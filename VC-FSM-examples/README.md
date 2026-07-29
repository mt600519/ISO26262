# VC Functional Safety Manager (VC FSM) —— Examples 学习笔记

面向 **ISO 26262** FMEDA 流程，对 Synopsys **VC Functional Safety Manager** 官方
`examples/` 目录逐文件分析整理的学习文档。

## 内容

| 文件 | 说明 |
|------|------|
| [`VC-FSM_examples学习笔记.md`](VC-FSM_examples学习笔记.md) | Markdown 源文档 |

## 覆盖范围

- VC FSM Client/Server 架构与数据模型（Design / Tech / SM Type / Synthetic Hierarchy / FMEA / FMEDA）
- 各示例用途与文件含义：
  - 完整流程 demo：`demo`、`HL-fRMEM`（高层概念 FMEDA）、`FaultInjection…`（故障注入闭环）、`DesignDataBatchImport`
  - 独立工具脚本：`IEC62380` / `SN29500`（基础失效率 BFR）、`ruleChecker`、`netlist_extract`、`quickSynthesis`、`Spyglass_FA`、`fdb`（SFF）
- IPData 核心表语义：Main/SM FMEA、Main/SM FMEDA、PSM（主安全机制）、LSM（潜伏安全机制）
- `DcKrf/DcKmpf` 经 `sm_type.csv` 换算成 **SPFM / LFM** 的完整推导（含 λ 分桶、汇总公式、数值示例）

> 依据 VC FSM `examples/` 实际文件与 ISO 26262-5 度量定义整理，供学习参考。
> 命令语法请以所用版本的 VC FSM User/Administrator Guide 为准。
