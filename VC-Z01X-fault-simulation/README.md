# VC‑Z01X 故障仿真学习手册

面向车规 ISP 芯片验证、用于 **ISO 26262 审查准备**的 Synopsys **VC‑Z01X** 故障仿真（Fault Simulation）学习文档。

## 内容

| 文件 | 说明 |
|------|------|
| [`VC-Z01X_故障仿真学习手册.md`](VC-Z01X_故障仿真学习手册.md) | Markdown 源文档 |
| `VC-Z01X_故障仿真学习手册.pdf` | 排版后的 PDF（含中文字体、表格样式） |

## 覆盖范围

- 故障模型：Stuck‑At / Transient（功能安全）、Transition / Bridge（制造测试）
- 注入与观测机制：Good/Faulty Machine、观测点 vs 诊断点（Observe/Detect）、UCFD 系统任务
- 故障分类：两字符状态码、内建状态组、Fault Dropping、UDFS 与 ISO 26262 映射
- 覆盖率：Test/Fault Coverage、Diagnostic Coverage、SPFM/LFM
- SFF 标准故障格式、FCM 命令、从零搭建完整流程
- 常见陷阱、审查红线、Q&A、工具置信度与 CoU/AoU

> 依据 Synopsys《VC Z01X User Guide, X‑2025.06》与《Z01X Manufacturing Assurance User Guide, X‑2025.06》整理。
> 命令语法与状态码请以所用版本 User Guide 正文与附录 A/B 为准。
