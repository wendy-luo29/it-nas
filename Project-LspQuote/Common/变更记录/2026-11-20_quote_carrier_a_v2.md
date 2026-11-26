---
change_ids: [CHG-002, CHG-002b]
module: lsp_quote
feature: pricing
date: 2026-11-20
type: carrier_version
---

# 2026-11-20：承运商 A v2.0 计费规则升级

## 变更概要
Carrier A 计费规则从 v1.2 升级到 v2.0，体积重公式变更：
- v1.2：`volumetric_weight = length * width * height / 5000`
- v2.0：`volumetric_weight = length * width * height / 4000`（更严格，重量更大）

同时新增偏远地区附加费（Remote Area Surcharge）。

## 具体变更
- v1.2 → v2.0 的体积重系数 5000 → 4000
- 新增 `remote_area_flag` 参数，指定是否触发附加费
- 附加费金额：固定 $3.50 / 包裹
- v2.0 上线时间：2026-12-01

## 影响用例（已登记 impact-index.yaml）
```
TC-QUOTE-CARA-001  test_pricing_normal_parcel
TC-QUOTE-CARA-015  test_volumetric_weight_calculation
TC-QUOTE-CARA-022  test_compare_price_against_carrier_b
TC-QUOTE-CARA-031  test_remote_area_surcharge
```

## 回归范围
- 所有 Carrier A 相关计费用例
- 比价择优逻辑（Carrier A vs Carrier B 结果会变）
- DB 写入字段（remote_area_flag 是新字段）

## 关联配置
- `config/lsp_quote/carrier_a/v1.2.yaml`（保留，历史版本）
- `config/lsp_quote/carrier_a/v2.0.yaml`（新增）
