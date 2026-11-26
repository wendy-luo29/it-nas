# 数据库关键表

## lsp_backend

### lsp_address_suburb_postcode
地址总表，按 state + postcode + suburb 联合查询。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| state_code | VARCHAR(10) | NOT NULL | 州/省缩写，如 "NSW" |
| postcode | VARCHAR(10) | NOT NULL | 邮编，字符串（保留前导零） |
| suburb_name | VARCHAR(100) | NOT NULL | 郊区/城市名 |
| suburb_code | VARCHAR(20) | — | 外部编码 |
| country_code | VARCHAR(5) | DEFAULT 'AU' | 国家代码 |

**注意**：suburb_name 查询大小写不敏感（应用层 UPPER() 包装）。

## lsp_quote

> 【待补充】Carrier A/B 计费相关表结构

## lsp_booking

> 【待补充】申报/下单相关表结构
