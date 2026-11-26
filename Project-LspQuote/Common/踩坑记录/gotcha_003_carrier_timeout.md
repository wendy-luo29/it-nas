# gotcha_003：承运商 API 超时配置差异

## 踩到什么坑
不同承运商的超时容忍度不一样：
- Carrier A：内部网关超时 30s，可以设置 `api_timeout: 30`
- Carrier B：外部 API 超时只有 10s，且偶有抖动

统一用 `api_timeout: 30` 配置导致 Carrier B 测试经常被 CI 误判为环境不稳。

## 正确做法
- 超时配置按承运商维度放在 `config/lsp_quote/carrier_X/vY.Z.yaml` 里
- 根配置 `api_timeout` 作为兜底值，具体承运商覆盖
- fixture 在 session 级加载对应承运商配置

## 关联
- 配置结构：`config/base.yaml` → `config/lsp_quote/carrier_a/v2.0.yaml`
- fixture：`tests/lsp_quote/conftest.py -> quote_client`
