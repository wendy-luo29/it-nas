# gotcha_001：PO Box 正则大小写不敏感 + 标点变体

## 踩到什么坑
PO Box 正则最初只覆盖了 "PO Box" 大写形式，漏掉了：
- "po box"（全小写）
- "Po Box"（首字母大写）
- "P.O.Box" / "P.O. Box"（带点号）
- "POST OFFICE BOX"（全称）

导致 test_pobox_case_variants xfail（原本应该 pass）。

## 正确做法
- PO Box 关键词匹配用 `re.compile(r'(?:p\.?o\.?|post\s*office)\s*(?:box|drawer)', re.IGNORECASE)`
- 测试数据在 `data/lsp_backend/address_validate.json -> pobox_positive` 中维护
- 每次新增 PO Box 写法时，同步更新数据 + 规则文档

## 关联
- 规则文档：[rules/lsp_backend/address_validation.md#R-6 PO Box 检测](../业务规则/lsp_backend/address_validation.md)
- 变更记录：CHG-001 之前（隐性知识）
