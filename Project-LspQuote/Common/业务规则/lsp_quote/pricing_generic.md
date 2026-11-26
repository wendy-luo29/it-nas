---
module: lsp_quote
feature: pricing
version: v1.0
status: draft
confirmed_by: ""
confirmed_date: ""
related_cases: []
related_gotchas: [gotcha_003]
---

# LspQuote 计费规则（通用）

> v1.0 draft by 产品 2026-11-18
> 【待确认】所有规则均为草稿，需产品和承运商确认

## 基本信息

- **接口**: 待定（Apifox 文档补充中）
- **所属模块**: LspQuote
- **承运商差异**: 每个承运商有独立计费公式和阈值
- **核心功能**: 根据包裹信息（重量、体积、目的地）匹配承运商并计算费用

## 计费核心公式（每个承运商变体）

```
final_price = base_fee + weight_fee + volumetric_fee + distance_fee - discount
```

每个承运商变体在 `config/lsp_quote/carrier_X/vY.Z.yaml` 中配置具体参数。

## 比价择优规则

【待确认】比价策略：
- 最低价格？
- 加权评分（价格 60% + 时效 30% + 可靠性 10%）？
- 用户指定优先承运商？

## 影响索引入口

详见 [impact-index.yaml](../../../impact-index.yaml) `module: lsp_quote` 相关条目。
