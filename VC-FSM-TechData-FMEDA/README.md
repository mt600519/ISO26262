# VC FSM Tech Data 建模报告 —— 28nm 车规集成 ISP

面向 **ISO 26262** FMEDA，针对 Synopsys **VC Functional Safety Manager** 的 Tech Data /
IEC 62380 BFR 设置，以一颗 **28nm CMOS 集成 ISP**（自研标准单元 ASIC）为对象整理的建模实操报告。

## 内容

| 文件 | 说明 |
|------|------|
| [`VC-FSM_TechData_IEC62380_28nm-ISP建模报告.md`](VC-FSM_TechData_IEC62380_28nm-ISP建模报告.md) | Markdown 源文档 |

## 覆盖范围

- **Tech Data** 是什么、多实例（instance）机制、`MappedDesignKey` 与 λ 计算
- **永久 vs 瞬态**两套独立模型（温度剖面只驱动永久；瞬态走 JESD89 SER）
- **IEC 62380 Table 16（λ1/λ2）器件族选型** —— "按设计类型而非功能"原则，10 项 Tech Data Type 的锁定映射
- **28nm 瞬态 SER 数据**（Flops/Latches/RAM/ROM/Flash/逻辑/模拟，含机制 A 设置）
- **IEC 62380 Parameters + Mission Profile** —— 结温模型、参数详情、车规工作/循环相位
- 可直接运行的 `set_bfr_setting` **tcl 脚本**
- **附录 A：四张 FMEA/FMEDA 表的 PSM/LSM 与 DcKrf/DcKmpf 填法速查** —— Main/SM × FMEA/FMEDA、两个 LSM 的关系、判定口诀、ISP 示例

> 依据 VC FSM `examples/` 实际 API、IEC TR 62380:2004、ISO 26262‑5/11:2018 整理。
> 文中失效率为行业典型量级估计，正式 FMEDA 须用 foundry/JESD89 实测与整车 mission profile 替换，并登记为 AoU。
> IEC/ISO 授权文档内容仅供内部学习建模，请勿再分发原始表页。
