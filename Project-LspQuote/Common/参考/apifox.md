# Apifox 接口文档

| 模块 | 功能 | Apifox 链接 | 版本 | 备注 |
|------|------|------------|------|------|
| LspBackend | address_validation | 待补充 | v2.1 | 当前线上版本 |
| LspBackend | strategy_config | 待补充 | v1.0 | 策略配置通用接口 |
| LspQuote | pricing (Carrier A) | 待补充 | v1.2 / v2.0 | v2.0 2026-12-01 上线 |
| LspQuote | pricing (Carrier B) | 待补充 | v1.0 | — |
| LspBooking | declare (Carrier A) | 待补充 | v1.0 | — |
| LspBooking | declare (Carrier B) | 待补充 | v1.0 | — |

## OpenAPI 规范要求

- 所有导出的 OpenAPI JSON 放在 `requirements/specs/<module>/` 下
- 文件名格式：`<feature>.v<major>.<minor>.<patch>.json`
- 同模块同功能不同版本并存，config 里显式指定版本号
