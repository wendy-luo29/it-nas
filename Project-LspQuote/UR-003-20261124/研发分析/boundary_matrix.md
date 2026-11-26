# REQ-003 燃油附加费 — 约束与边界矩阵

## 一、费率事实表（来源：需求《分析》表格；DB 实测待校准）

| 渠道 | carrier_id | carrier_code | 旧 fuel_rate | 新 fuel_rate | 变化 |
|---|---|---|---|---|---|
| TGE | CAR1772558358570467347 | UBI.AU2AU.IPEC | 30.21% | 29.23% | -0.98pp |
| TGE | CAR2025062611067266788 | AU.TGE.PRIORITY.PRO | 23.74% | 28.61% | +4.87pp |
| Allied Express | CAR1770625562872602624 | aoe | 26.79% | 43.5%（26.9%+16.6%） | +16.71pp |
| Allied Express | CAR1020261008100000001 | aoe | 同 code | 43.5% | 同 code |
| Hunter Express | CAR1770625562872602626 | road_freight | 29%；WA 36% | 29%；WA 36% | 不变 |
| Hunter Express | CAR1020261008100000002 | road_freight | 同 code | 同 code | 不变 |
| Australia Post | CAR1770610202510160001~0004 | AUPOST.N/Y.E-Parcel/Express-Post | 13.5% | 13.5% | 不变 |
| DFE | CAR1772558358570467350 | direct_freight | 空 | 33.1%（接口） | 接口计费 |

> 实测日期：未实测（来自需求截图）。DB 用例以 `lsp_carrier` 查询；若燃油费实际维护在 `lsp_carrier_rate`，以该表为准。

## 二、用例分布（按承运商拆分）

| 承运商 / code | 数据库检查 | 计费接口检查 | 期望燃油费 |
|---|---|---|---|
| TGE / UBI.AU2AU.IPEC | TC-FUEL-001 | TC-FUEL-010 | 29.23% |
| TGE / AU.TGE.PRIORITY.PRO | TC-FUEL-002 | TC-FUEL-011 | 28.61% |
| Allied Express / aoe（两个 id） | TC-FUEL-003 | TC-FUEL-012 | 43.5% |
| Hunter Express / road_freight（两个 id） | TC-FUEL-004 | TC-FUEL-013 | 29%；WA 36% |
| Australia Post / AUPOST.N.E-Parcel | TC-FUEL-005 | TC-FUEL-014 | 13.5% |
| Australia Post / AUPOST.N.Express-Post | TC-FUEL-006 | TC-FUEL-015 | 13.5% |
| Australia Post / AUPOST.Y.E-Parcel | TC-FUEL-007 | TC-FUEL-016 | 13.5% |
| Australia Post / AUPOST.Y.Express-Post | TC-FUEL-008 | TC-FUEL-017 | 13.5% |
| DFE / direct_freight | TC-FUEL-009 | TC-FUEL-018 | 接口 33.1% |

## 三、约束矩阵

| 约束ID | 维度 | 约束 | 数据点 | 用例ID | 状态 |
|---|---|---|---|---|---|
| BC-001 | DB | UBI 新费率 29.23% | 30.21% / 29.23% | TC-FUEL-001 | 已覆盖 |
| BC-002 | DB | PRO 新费率 28.61% | 23.74% / 28.61% | TC-FUEL-002 | 已覆盖 |
| BC-003 | DB | aoe 合计 43.5%，两个 id 一致 | 26.79% / 43.5% | TC-FUEL-003 | 已覆盖，存储方式待确认 |
| BC-004 | DB | Hunter 29%、WA 36% 不变 | 29% / 36% | TC-FUEL-004 | 已覆盖 |
| BC-005 | DB | 澳邮四个 code 均 13.5% | 13.5% ×4 | TC-FUEL-005~008 | 已覆盖 |
| BC-006 | DB | DFE 接口计费 33.1%，不重复配置 | 33.1% | TC-FUEL-009 | 已覆盖 |
| BC-007 | DB | 其他字段不受影响 | code / 名称 / 状态等全字段对比 | TC-FUEL-001~009 | 已覆盖 |
| BC-008 | 接口 | 变更承运商按新费率计费 | 1.2923 / 1.2861 / 1.435 | TC-FUEL-010~012 | 已覆盖 |
| BC-009 | 接口 | 未变更承运商按现行费率计费 | 1.29 / 1.36 / 1.135 / 1.331 | TC-FUEL-013~018 | 已覆盖 |
| BC-010 | 接口 | 每个承运商都产出报价并参与择优 | 9 个 code | TC-FUEL-010~018 | 已覆盖 |
| BC-011 | 部署 | 更新 SQL 只涉及 11 条承运商记录 | 11 个 carrier_id | TC-FUEL-019 | 已覆盖 |
| BC-012 | 异常 | 0% / NULL / 负值 / >100% | — | 未覆盖 | 本次范围外，行为待产品确认 |
| BC-013 | 时间 | 生效时点前后取值 | — | 未覆盖 | 需求未给生效日期，本次范围外 |

## 四、维度覆盖表

| 类别 | 是否覆盖 | 约束ID / 用例 | 缺失原因 |
|---|---|---|---|
| 数据库费率值 | 是 | BC-001~006；TC-FUEL-001~009 | — |
| 其他字段隔离 | 是 | BC-007；TC-FUEL-001~009 | — |
| 接口金额计算 | 是 | BC-008/009；TC-FUEL-010~018 | — |
| 每个承运商均有计费 | 是 | BC-010；TC-FUEL-010~018 | — |
| 择优结果 | 是 | TC-FUEL-010~018 | 每条计费用例均包含与其他承运商一起询价的择优检查 |
| 发布 SQL 范围 | 是 | BC-011；TC-FUEL-019 | — |
| 生效时间切换 | 否 | BC-013 | 需求未给生效日期 |
| 异常费率值 | 否 | BC-012 | 本轮范围外 |
