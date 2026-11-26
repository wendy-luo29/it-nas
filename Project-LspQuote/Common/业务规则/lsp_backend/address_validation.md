---
module: lsp_backend
feature: address_validation
version: v2.1
status: active
confirmed_by: 业务方
confirmed_date: 2026-11-15
source_requirement: UR-001-20261115/接口文档/all.v2.1.0.json
related_cases: [TC-ADDR-001, TC-ADDR-002, TC-ADDR-003]
related_gotchas: [gotcha_001, gotcha_002]
---

# 地址校验规则

> v2.1 confirmed by 业务方 2026-11-15
> 来源：config/env.yaml address_validation 区块 + 测试代码注释

## 基本信息

- **接口**: `POST /address/validate`
- **所属模块**: LspBackend
- **承运商差异**: 无（通用地址校验服务，所有承运商共用）
- **当前版本**: v2.1
- **国家范围**: 仅澳大利亚（AU），其他国家拒绝

## 业务规则（confirmed）

### R-1 存在性判断
- `existence = state + postcode + suburb` 在地址总表中联合查询
- 命中 → `data.success=true`
- 未命中 → `HTTP 200, data.success=false, reason="address not found"`
- 注意：**not-found 走 HTTP 200，不是 404**

### R-2 address/address2 用途
- 仅用于 PO Box 检测，不影响存在性判断
- address 为空或 null 时跳过 PO Box 检测

### R-3 state 字段
- 接受缩写（如 "NSW"）或全名（如 "New South Wales"）
- 全名会被标准化为缩写存入 `data.stateCode`

### R-4 country 字段
- 接受 "AU" / "Australia"（大小写不敏感）
- 其他国家 → HTTP 400 或业务失败 `data.success=false`
- 默认值：Australia（country 缺失或空字符串时）
- 【待确认】null 时是否等同空字符串

### R-5 is_return 字段
- 标记是否为退货地址
- 【待确认 v2.2】is_return=true 时 adviceAddress 生成规则

### R-6 PO Box 检测
- address 或 address2 包含 PO Box 关键词 → `isPoBox=true`
- 关键词：P.O. Box, PO Box, Post Office Box, GPO, LPO 等
- 【踩坑 gotcha_001】正则大小写不敏感，但空格/标点变体需要覆盖

### R-7 响应结构（成功时）
```json
{
  "status": 200,
  "code": "200",
  "msg": "success",
  "data": {
    "success": true,
    "stateCode": "NSW",
    "suburbCode": "...",
    "suburb": "...",
    "postcode": "...",
    "countryCode": "AU",
    "isPoBox": false,
    "adviceAddress": "",
    "orderId": "...",
    "address": "...",
    "address2": "...",
    "fullAddress": "..."
  }
}
```
- 【待确认】OpenAPI 未声明 required，data 是否始终返回 15 个字段

### R-8 异常响应
| 场景 | HTTP | 行为 |
|------|------|------|
| DB 超时/不可用 | — | error response（never success=true） |
| 必填缺失/类型错误 | 400 | msg 含原因，data=null |
| 国家非 AU | 200 或 400 | 取决于具体错误类型 |

### R-9 幂等与并发
- 相同入参多次调用 → 结果一致（success / reason / isPoBox / stateCode 完全相同）
- 并发 10 路无串号、无竞态

## 数据库依赖

- **表**: `lsp_address_suburb_postcode`
- **查询**: `WHERE state_code = ? AND postcode = ? AND UPPER(suburb_name) = UPPER(?) LIMIT 1`
- **注意**: suburb 大小写不敏感；postcode 前导零必须保留字符串类型
- 【踩坑 gotcha_002】DB 字段 NOT NULL 但 API 可能返回空串

## 鉴权

- 无需鉴权（内部服务）
- 网络层有 API Gateway 限流保护
