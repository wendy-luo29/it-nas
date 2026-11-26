# 术语表

| 术语 | 全称 | 含义 |
|------|------|------|
| LSP | Logistics Service Provider | 物流服务提供商，本项目的被测系统 |
| LspBackend | — | 基础信息配置模块（地址校验、策略配置） |
| LspQuote | — | 物流计费/报价模块 |
| LspBooking | — | 物流申报/下单模块 |
| AC | Atomic Capability | 原子能力（需求拆解产物，设计用例的锚点） |
| TC | Test Case | 测试用例 |
| PO Box | Post Office Box | 邮政信箱，特殊地址类型 |
| cube weight | volumetric weight | 体积重，长*宽*高/4000,四舍五入保留3位小数|
| shipping weight | — |根据承运商的配置来判断，lsp_carrier表的 weight_type字段，100：净重  200：体积重  300：发运重
| carrier | — | 承运商，如 Carrier A、Carrier B |
| RAS | Remote Area Surcharge | 偏远地区附加费 |
| Au | Australia | 澳大利亚（唯一支持的国家） |
| 澳邮承运商 | 计算pobox地址的承运商 | 根据地址校验(包含address1和address2）是否为pobox地址，如果是计费只能为carrier表中标记is_pobox=200的承运商计算，如果当前计费任务的策略或者指定承运商不包含澳邮承运商则计费失败