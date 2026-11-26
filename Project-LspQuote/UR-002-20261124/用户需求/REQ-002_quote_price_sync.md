---
requirement_id: REQ-001
version: v1.0
change_type: carrier_onboarding
carrier: AU.TGE.PRIORITY.PRO
carrier_id: CAR2025062611067266788
source: Apifox 导出 OpenAPI 3.0.1（LspQuote / QuoteTaskController）
original_file: all.v1.0.0.json
updated_at: 2026-11-24
owner: wendy-luo29
---

# 新增承运商 AU.TGE.PRIORITY.PRO 接入报价计费（POST /quote/price_sync）

> **变更类型：新增承运商接入（carrier_onboarding）**
> 本次需求**不是新增接口**：`POST /quote/price_sync` 是存量接口。本文档保留完整接口契约用于回归（baseline），本次验收范围仅限新增承运商 AU.TGE.PRIORITY.PRO 的计费接入（acceptance）。

## 1. 接口概览

| 项 | 内容 |
|---|---|
| 接口名称 | 承运商运费计算同步接口（存量） |
| 变更类型 | **新增承运商接入（carrier_onboarding）**，不是新增接口 |
| 请求方法 | POST |
| 接口路径 | `/quote/price_sync` |
| 接口说明 | 存量接口：按订单与包裹信息返回各承运商的运费报价列表；本次新增承运商 AU.TGE.PRIORITY.PRO 接入该链路 |
| 接口分组 | `test/LspQuote/quote-task-controller/QuoteTaskController` |
| 请求类型 | `application/json` |
| 响应类型 | `application/json` |
| 鉴权方式 | **本接口不经网关，不做鉴权**（实测请求头无 `Authorization`）；不返回 401 |
| 幂等说明 | **本接口不做幂等**：每次调用都会重新计算并生成新的计费任务，重复提交不会复用上一次结果。`requestId` 仅作请求追踪（第三方接入时必传），不参与去重；响应不回显 `requestId`/`orderCode`/`quoteTaskId`，用例每次使用唯一 `quoteTaskId` 查库定位本次任务。**本接口暂不支持多仓**，`warehouse` 数组只传 1 个仓 |
| 调用方 | ERP 下单或运费试算流程 |
| 调用频率 | 随订单量触发 |

### 1.1 本次范围

| 范围 | 内容 |
|---|---|
| **acceptance（本次验收）** | AU.TGE.PRIORITY.PRO 能被策略/VIR 选中；按 PRO 价卡/重量段/fuel/zone 计费；仅支持 calcMode=1；箱规与过滤；与存量承运商比价择优；PRO 任务正确落库；未知仓库 / 不存在的 carrierId → HTTP 500 |
| **baseline（回归，不作为本次新增验收）** | `/quote/price_sync` 的请求/响应契约、鉴权、幂等、单仓、错误码框架、参数校验、推送参数、通用落库口径；缺必填 400 / carrierIds 优先级 / pushResultTo=100 / 无收件区域 |

> **发布口径**：acceptance 用例全部通过（skip/xfail 不算通过），baseline 回归无新增失败。未知仓库、不存在的 carrierId 未修复前不可发版。

> **以下 §2-§6、§8、§9 为存量接口基线（baseline）**，用于回归；本次验收项见 §7.2 的验收口径。

## 2. 请求定义

### 2.1 请求头

| 字段 | 业务必填 | 实测值 | 说明 |
|---|---|---|---|
| Content-Type | 是 | `application/json` | 请求体类型 |
| Accept | 否 | `*/*` | 客户端可接受类型 |
| Authorization | 否 | 未携带 | 本接口不经网关，不使用 `Authorization` |
| User-Agent | 否 | `Apifox/1.0.0 (https://apifox.com)` | 客户端自动生成，不参与业务校验 |
| Host | 否 | `192.168.1.71:13100` | 目标主机，由客户端/网关生成 |
| Accept-Encoding | 否 | `gzip, deflate, br` | 传输编码协商 |
| Connection | 否 | `keep-alive` | 连接复用 |
| Content-Length | 否 | 示例 1156 | 由 HTTP 栈计算 |

> 实测来源：Apifox「实际请求」Header（2026-11-23，`POST http://192.168.1.71:13100/quote/price_sync`，HTTP 200）。除 `Content-Type` 外，其余 Header 由客户端/网关生成；当前直连环境**没有 `Authorization`**。

### 2.2 请求体（QuoteTaskDto）

| 字段 | 类型 | 必填 | 约束 | 默认值 | 说明 |
|---|---|---|---|---|---|
| requestId | string | 否 | **max 64**（DB `lsp_quote_task.request_id varchar(64)`） | — | 请求追踪 ID；**不参与幂等**；本接口可空，第三方接入场景必传 |
| ownerId | string | 是 | **max 32**（DB `owner_id varchar(32)` NOT NULL） | — | 归属人 ID |
| orderCode | string | 是 | 最大长度 32（JSON + DB `order_code varchar(32)` 一致） | — | ERP 销售订单号 |
| carrierStrategyCode | string | 条件必填 | **DB `carrier_strategy_code TEXT` 无硬长度上限**；取值两种形式：① 指定策略编码（如 `quote_price`）② 指定承运商 `VIR\|carrierId\|carrierId`。**承运商选择：本字段与 `carrierIds` 至少传一个；仅传本字段时按策略选承运商，仅传 `carrierIds` 时按列表选承运商，两者同传时以 `carrierIds` 为准；两者均未传或为空、`VIR` 段非法、carrierId 不存在、策略绑定空集合、策略编码不存在均返回 500** | — | 物流策略编码 |
| carrierIds | array\<string\> | 否 | **每项 max 32**（DB `lsp_carrier.id varchar(32)`） | — | 指定承运商 ID 列表；**与 `carrierStrategyCode` 同传时以本字段为准**，`carrierStrategyCode` 仅在本字段未传时生效 |
| shipmentType | string | 否 | **max 32**（DB `shipment_type varchar(32)`）；**不校验枚举值** | — | 计费流水号分类，仅透传 |
| shipTo | object | 是 | 见 2.3 | — | 派件地信息 |
| warehouse | array\<object\> | 是 | 见 2.4 | — | 发件地信息 |
| isPushWms | integer | 否 | 100 推送 \| 200 不推送 | 200 | 是否推送结果到终端；为 100 时按 `pushResultTo` 推送（0 表示不推送） |
| pushResultTo | integer | 否 | 0 不推送 \| 301 SCP \| 200 WMS \| 300 ERP；**接口直调场景传 0**，301/200/300 仅业务流程链路使用（100 为之前业务的 SCP 旧值，兼容保留：传 100 等同 301 推送 SCP，不报错） | 0 | 设置结果推送 |
| shippingType | integer | 否 | **不校验枚举值**（已知 100 发货单 / 200 退货单 / 300 手工单 / 600 其他单） | 100 | 发货单类型，仅透传，不参与计费 |
| closePoboxCheck | integer | 否 | 100 关闭 \| 200 不关闭 | 200 | 透传字段；PO Box / 澳邮过滤规则不在本接口范围（见对应功能说明） |
| bookingOrderId | string | 否 | 申报运费计费时传入（仅业务流程链路触发；接口级测试不覆盖该分支） | — | 申报单号 |

### 2.3 派件地址（shipTo / QuoteTaskAddressDto）

> DB 映射：落库 `lsp_quote_task_address`（type=派件方向）；zone 匹配查 `lsp_carrier_zone`

| 字段 | 类型 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| country | string | 是 | **max 32**（DB `country varchar(32)` NOT NULL）；**仅支持 AU** | 国家（兼容大小写及全称） |
| state | string | 是 | **max 32**（DB `state varchar(32)` NOT NULL） | 州简称；程序兼容全称，**落库为简称** |
| suburb | string | 是 | **max 255**（DB `suburb varchar(255)` NOT NULL） | 城区 |
| postcode | string | 是 | **max 32**（DB `postcode varchar(32)` NOT NULL） | 邮编 |
| address1 | string | 是 | **max 255**（DB `address1 varchar(255)` NOT NULL） | 地址第一行 |
| address2 | string | 否 | **max 255**（DB `address2 varchar(255)` NOT NULL DEFAULT=''，接口可空） | 地址第二行 |

### 2.4 发件地（warehouse[] / QuoteTaskWarehouseDto）

> 发件地有两层含义：① 仓库编码引用 `lsp_warehouse.code`（已建好的仓库主数据）；② 发件地址快照（同 shipTo 结构）。DB 侧 `lsp_quote_task` 只存 `warehouse_code`，地址字段不入库。
>
> **本接口暂不支持多仓：`warehouse` 数组长度固定为 1。**

| 字段 | 类型 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| warehouseCode | string | 是 | **max 50**（DB `lsp_warehouse.code varchar(50)`） | 仓库编码（引用 `lsp_warehouse` 主数据，当前为 BG03\|BG05\|BG06\|BG07\|BG08\|BGS1；未建仓编码按未知仓库处理 → 计费失败，§9.2） |
| shipmentCode | string | 是 | **max 64**（DB `lsp_quote_task.shipment_code varchar(64)` NOT NULL） | 发货单号 |
| country | string | 是 | **max 32**（同 shipTo） | 国家（发件地快照，非 DB 强约束） |
| state | string | 是 | **max 32**（同 shipTo） | 州 / 省 |
| suburb | string | 是 | **max 255**（同 shipTo） | 城区 |
| postcode | string | 是 | **max 32**（同 shipTo） | 邮编 |
| address1 | string | 是 | **max 255**（同 shipTo） | 地址第一行 |
| address2 | string | 否 | **max 255**（同 shipTo） | 地址第二行 |
| packages | array\<object\> | 是 | 见 2.5 | 该仓库发出的包裹 |

### 2.5 包裹（packages[] / QuoteTaskPackageDto）

> DB 映射：`lsp_quote_task_package`

| 字段 | 类型 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| packageCode | string | 是 | **max 32**（DB `package_code varchar(32)` NOT NULL）；**同一请求内唯一，重复按参数校验失败（HTTP 500）** | 包裹编码 |
| items | array\<object\> | 是 | 见 2.6 | 包裹内 SKU 明细 |
| length | number | 否 | **DB 精度 `double(12,3)`**，默认 0（DB DEFAULT=0.000） | 包裹长度；缺省时系统取 SKU 汇总或自动组包 |
| width | number | 否 | **DB 精度 `double(12,3)`**，默认 0 | 包裹宽度 |
| height | number | 否 | **DB 精度 `double(12,3)`**，默认 0 | 包裹高度 |
| weight | number | 否 | **DB 精度 `double(12,3)`**，默认 0 | 包裹重量 |

> 包裹层 `length/width/height/weight` 允许全为 0（表示回退到 SKU 层自动组包）；**四个字段必须同时为 0 或同时 >0，部分为 0 按参数校验失败（HTTP 500）**。业务暂未设包裹数上限，**用例设计按单请求最多 20 个包裹构造**。

### 2.6 包裹内明细（items[] / QuoteTaskPackageSkuDto）

> DB 映射：`lsp_quote_task_package_sku`

| 字段 | 类型 | 必填 | 约束 | 说明 |
|---|---|---|---|---|
| sku | string | 是 | **max 40**（DB `sku varchar(40)` NOT NULL） | SKU 编码 |
| qty | integer | 是 | 最小 1（JSON schema）；**DB `smallint` NOT NULL** | 数量；**组包时 qty>1 按 qty 个相同 SKU 参与计算** |
| length | number | 是 | 最小 0.01；**DB 精度 `double(12,3)`** | SKU 长度 |
| width | number | 是 | 最小 0.01；**DB 精度 `double(12,3)`** | SKU 宽度 |
| height | number | 是 | 最小 0.01；**DB 精度 `double(12,3)`** | SKU 高度 |
| weight | number | 是 | 最小 0.01；**DB 精度 `double(12,3)`** | SKU 重量 |
| cube | number | 否 | **DB 精度 `double(12,3)` NOT NULL** | 体积，填写，落库由系统计算；**不参与计价** |
| cubeWeight | number | 否 | **DB 精度 `double(12,3)` NOT NULL** | 体积重，填写，落库由系统计算；**不参与计价（计费重仅取称重，见 §7）** |

## 3. 响应定义

### 3.1 响应包装（RestResultListQuotePriceSyncVo）

| 字段 | 类型 | 说明 |
|---|---|---|
| data | array\<QuotePriceSyncVo\> | 报价结果列表,排序：isBest=1 优先，其他按 `shippingCost` 升序 |
| code | string | 业务状态码 |
| msg | string | 提示信息 |
| tc | integer(int64) | 框架保留字段（耗时/时间戳），当前无业务语义，用例不断言 |
| tp | integer | 框架保留字段，当前无业务语义，用例不断言 |
| hasMore | integer | 同步接口保留字段，当前无业务语义，用例不断言其值 |
| success | boolean | 是否成功 |

> **同步响应不返回 `requestId` / `orderCode`**（2026-11-23 业务方确认）：这两个是请求级标识，仅随请求传入，落库到 `lsp_quote_task.request_id` / `lsp_quote_task.order_code`。需要校验时按入参值查库定位本次任务，不在响应中断言。报价行的唯一标识见 §3.2。

### 3.2 报价项（QuotePriceSyncVo）

> DB 映射：承运商主数据来自 `lsp_carrier` + `lsp_carrier_agent` + `lsp_carrier_channel`；计算结果对应 `lsp_quote_task_` / `lsp_quote_task_job_log`；**无独立的 QuotePriceSyncVo 落库表**。

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| calcMode | integer | — | 计算方式：1=一票一件；2=一票多件（DB `lsp_quote_task_job.calc_mode smallint`） |
| canShipping | boolean | — | 是否可承运。承运商价卡会同时计算一票一件和一票多件；若承运商仅支持一票一件，一票多件结果标记为 false 并跳过该计算方式。**两种方式的原始结果写入 `lsp_quote_task_job_log.result_data`；接口响应只返回有计费结果的承运商，canShipping=false 的行不出现在 data[] 中；响应中出现的行 `canShipping` 恒为 `true`；新增承运商仅支持一票一件，响应中该承运商行 `calcMode` 恒为 1** |
| agentCode | string | — | **max 255**（DB `lsp_carrier_agent.code varchar(255)` NOT NULL） |
| agentName | string | — | **max 128**（DB `lsp_carrier_agent.name varchar(128)` NOT NULL） |
| channelCode | string | — | **max 255**（DB `lsp_carrier_channel.code varchar(255)` NOT NULL） |
| channelName | string | — | **max 128**（DB `lsp_carrier_channel.name varchar(128)` NOT NULL） |
| carrierCode | string | — | **max 255**（DB `lsp_carrier.code varchar(255)` NOT NULL） |
| carrierName | string | — | **max 128**（DB `lsp_carrier.name varchar(128)` NOT NULL） |
| bookingCarrierId | string | — | **max 32**（DB `lsp_carrier.id varchar(32)` NOT NULL） |
| bookingCarrierCode | string | — | 下游申报使用的唯一编码（同 carrierCode） |
| bookingCarrierCalc | string | — | 申报承运商的申报方式（**string 枚举**）：`"1"`=一票一件；`"2"`=一票多件；与 `calcMode`（integer）取值对应（DB 无直接映射，运行时决定） |
| remark | string | — | 备注 |
| shippingCost | number | — | 比价择优的运费，精度：四舍五入保留三位小数；**DB `predict_price decimal(12,3)`** |
| choiceShippingCost | number | — | （暂未启用，返回 null） |
| shippingCostWithTax | number | — | 暂未启用，返回 null |
| surcharge | number | — | 暂未启用，返回 null |
| closePoboxCheck | integer | — | 透传字段：100=关闭；200 或为空=不关闭（本接口不做 PO Box 过滤，DB `lsp_quote_task.close_pobox_check smallint`） |
| isBest | integer | 2 | 最优标记：1=全局最优（最终运费），**全局唯一，恰 1 行**；2=内部最优（承运商内部一票一件 vs 一票多件择优）。见 §7 择优规则 |

> **报价行唯一标识**（用于定位具体一条报价、断言归属）：
> - **本接口不做幂等**：每次调用生成新的 `quote_task_id`；用例按 `request_id` / `quote_task_id` 定位本次结果，不要用 `orderCode` 去重
> - 行级唯一键：`bookingCarrierId + calcMode`（同一承运商同一 calcMode 只有一条 isBest=2 的内部最优行）
> - 在同一个包装内，**不会出现** `bookingCarrierId` 相同的多条 data[] 行（两层择优后每承运商只保留内部最优）
> - 被过滤的承运商（无价卡 / 无收件区域 / 超箱规的新增承运商 / 不可承运）不出现在 data[] 中

## 4. 请求示例

> 场景：BG03 仓库（Melbourne VIC 3012）发 1.15kg SKU 到 Sydney NSW 2000，询价 AU.TGE.PRIORITY.PRO 承运商。仓库、地址、承运商 ID 均来自 `lsp_scp` 库真实数据。

```json
{
  "requestId": "SAC202611190001",
  "ownerId": "BG",
  "orderCode": "TESTSO20261119-001",
  "carrierStrategyCode": "VIR|CAR2025062611067266788|CAR1772558358570467347", /或者为指定策略，例如 quote_price
  "shipTo": {
    "country": "AU",
    "state": "NSW",
    "suburb": "Haymarket",
    "postcode": "2000",
    "address1": "456 George Street",
    "address2": ""
  },
  "warehouse": [
    {
      "warehouseCode": "BG03",
      "shipmentCode": "TESTBG0320261119-001",
      "country": "AU",
      "state": "VIC",
      "suburb": "Maidstone",
      "postcode": "3012",
      "address1": "C 35-77 Emu Road",
      "packages": [
        {
          "packageCode": "QTP38177618321238-001",
          "items": [
            {
              "sku": "SKU-WIDGET-AU-001",
              "qty": 1,
              "length": 60.5,
              "width": 30,
              "height": 15,
              "weight": 1.15,
              "cube": 0.027,
              "cubeWeight": 6.806
            }
          ],
          "length": 60.5,
          "width": 30,
          "height": 15.0,
          "weight": 1.15
        }
      ]
    }
  ],
  "isPushWms": 200,
  "pushResultTo": 0,
  "shippingType": 100,
  "closePoboxCheck": 200
}
```

**示例说明**（与 DB 校验）：

| 项 | 值 | DB 依据 |
|---|---|---|
| 仓库 BG03 | `warehouse_code=BG03`, `address1="C 35-77 Emu Road"`, `state=VIC`, `postcode=3012` | `lsp_warehouse` 表 |
| 派件地址 | `suburb_name="haymarket"`, `postcode=2000`, `state_code=NSW` | `lsp_address_suburb_postcode` 表 |
| 承运商策略码 | `VIR\|CAR2025062611067266788\|CAR1772558358570467347`（PRO / 非 PRO） | `lsp_carrier.id` |
| 箱规 | 60.5×30×15cm，三边之和 105.5cm ≤ 120cm，称重 1.15kg ≤ 22kg | 通过箱规校验 |

## 5. 响应示例

> 两个承运商同时询价的两层择优结果。重量 1.15kg → 向上取整 2kg；zone 映射：MEL → ADL（需按 carrier_zone 表映射；派件地址 suburb+state+postcode 对应的 zone 为 lsp_carrier_zone 数据库实查值，非按目的城市名称推断），对应价卡段均为 `1.001 ≤ w ≤ 3.0`。
>
> **计费公式**：`shippingCost = fee × (1 + fuel_rate)`，四舍五入保留三位小数。DB 里存在的 `extra_fee` 列当前不参与计算，也无 RAS 类附加费（详见 §7 附加费规则）。

| 承运商 | fee | fuel_rate | 计算 | shippingCost | 层次 |
|---|---|---|---|---|---|
| UBI.AU2AU.IPEC | 7.87 | 0.097 | 7.87 × 1.097 | **8.633** | isBest=1 ✅ 全局最优 |
| AU.TGE.PRIORITY.PRO | 15.52 | 0.2861 | 15.52 × 1.2861 | **19.960** | isBest=2（内部最优） |

```json
{
  "code": "200",
  "msg": "success",
  "success": true,
  "tc": 1200,
  "tp": 0,
  "hasMore": 0,
  "data": [
    {
      "calcMode": 1,
      "canShipping": true,
      "agentCode": "ubi",
      "agentName": "UBI",
      "channelCode": "toll",
      "channelName": "Team Global Express",
      "carrierCode": "UBI.AU2AU.IPEC",
      "carrierName": "UBI Team Global Express Standard",
      "bookingCarrierId": "CAR1772558358570467346",
      "bookingCarrierCode": "UBI.AU2AU.IPEC",
      "bookingCarrierCalc": "1",
      "remark": "MEL->ADL / 1.001~3.0kg / fee=7.87 fuel=0.097",
      "shippingCost": 8.633,
      "choiceShippingCost": null,
      "shippingCostWithTax": null,
      "surcharge": null,
      "closePoboxCheck": 200,
      "isBest": 1
    },
    {
      "calcMode": 1,
      "canShipping": true,
      "agentCode": "ubi",
      "agentName": "UBI",
      "channelCode": "toll",
      "channelName": "Team Global Express",
      "carrierCode": "AU.TGE.PRIORITY.PRO",
      "carrierName": "UBI Team Global Express Priority",
      "bookingCarrierId": "CAR2025062611067266788",
      "bookingCarrierCode": "AU.TGE.PRIORITY.PRO",
      "bookingCarrierCalc": "1",
      "remark": "MEL->ADL / 1.001~3.0kg / fee=15.52 fuel=0.2861",
      "shippingCost": 19.960,
      "choiceShippingCost": null,
      "shippingCostWithTax": null,
      "surcharge": null,
      "closePoboxCheck": 200,
      "isBest": 2
    }
  ]
}
```

### 两层择优过程

```
请求输入     各承运商内部择优           全局择优后 data
─────────   ──────────────────         ────────────────
Carrier A   calcMode=1  $8.633  ←最优 → isBest=1  全局最低
Carrier A   calcMode=2  $8.633  (同1)   (合并为内部最优)
Carrier B   calcMode=1  $19.960  ←最优 → isBest=2  内部最优但全局第二
Carrier B   calcMode=2  $19.960  (同1)   (合并为内部最优)
```

> 当某承运商 `canShipping=false`（如一票多件超箱规）时，该 calcMode 不参与本承运商内部择优；承运商若无有效 calcMode 结果则整体不出现在响应中。
> 当某承运商只支持一票一件计费时，响应只返回一票一件结果；一票多件的计算结果与 canShipping=false 标记写入 `lsp_quote_task_job_log.result_data`，不在接口响应中返回。

## 6. 错误码

| code（string） | HTTP | message | 触发条件 | 处理建议 |
|---|---|---|---|---|
| `"200"` | 200 | `success` | 计算成功，返回择优结果和其他承运商计算明细 | — |
| `"400"` | 200 | 错误信息（非空，文案不校验，形如 `<字段名> must not null`） | 参数校验失败——spec 必填字段未传（`ownerId` / `orderCode` / `shipTo` / `warehouse` 及嵌套必填子字段，null/缺失）；字段传空串或非法枚举按 `"500"`（见下行与 §2.2） | 断言 HTTP=200、`code="400"`、`msg` 非空；进入计费前拦截，不落任务行（§9.3.2） |
| `"500"` | 500 | 错误信息（非空，文案不校验） | 参数校验失败（受校验字段的非法枚举值（如 `isPushWms`/`pushResultTo`）、长度/类型越界、同一请求 `packageCode` 重复、包裹层尺寸/重量部分为 0）、`carrierStrategyCode` 与 `carrierIds` 均未传或为空/`VIR` 段非法/carrierId 不存在/策略绑定空集合/策略编码不存在、计算失败、超时、无可用价卡、未知仓库/找不到发运地址 | 参数校验类看接口日志；计费失败原因查 `lsp_quote_task_job_log.failed_reason` |

**失败响应示例（HTTP 200，缺必填参数校验）**

```json
{
  "code": "400",
  "msg": "ownerId must not null",
  "success": false,
  "data": null
}
```

**失败响应示例（HTTP 500，非缺必填类参数校验）**

```json
{
  "code": "500",
  "msg": "invalid parameter: packageCode duplicated",
  "success": false,
  "data": null
}
```

> `msg` 只要求非空错误信息；用例断言 `code`（缺必填=`"400"`，其余=`"500"`）且 `msg` 非空，**不断言具体文案**。

## 7. 业务规则

> 范围：7.1 为存量接口通用规则（baseline 回归）；7.2 为本次新增承运商 AU.TGE.PRIORITY.PRO 的专属计费规则（acceptance 验收）。两者冲突时以 7.2 为准。

### 7.1 接口通用规则（baseline）

#### 流程

1. 接收上游订单、发件仓与包裹 / SKU 明细。
2. 按 `carrierStrategyCode`（或仅传 `carrierIds`）选承运商：① 指定策略编码（如 `quote_price`，策略绑定承运商集合）② 指定承运商（格式 `VIR|carrierId|carrierId`）③ 仅传 `carrierIds` 时按列表选承运商。若同时传 `carrierIds` 和 `carrierStrategyCode`，以 `carrierIds` 为准；两者均未传或为空、策略编码不存在返回 500。
3. 逐个承运商计算运费，返回报价数组。PO Box / 澳邮过滤**不在本接口范围**，仅透传 `closePoboxCheck`，不参与计费与过滤。
4. 承运商比价择优规则（`choiceShippingCost` / `shippingCostWithTax` / `surcharge` 暂未启用，均返回 null）
   - **第一层：承运商内部择优** — 每个承运商同时计算 calcMode=1（一票一件）和 calcMode=2（一票多件），取两者 `shippingCost` 较低者，标记为该承运商的**内部最优**（isBest=2）。若某承运商仅支持其中一种 calcMode，直接取该模式结果。
   - **第二层：全局择优** — 所有承运商的内部最优集合中，取 `shippingCost` 最低者标记为**全局最优**（isBest=1），即最终运费值。
   - `canShipping=false` 的计算方式不参与本承运商内部择优；该承运商若无有效计算结果则整体不出现（data[] 中不含该承运商行，也不含 null 元素）。
   - **过滤规则**：无价卡、无收件区域、超箱规的新增承运商、不可承运的承运商不进响应；无价卡 / 无收件区域过滤后若没有任何有效承运商，则整单计费失败（§6 500）；超箱规按本条「超箱规」规则处理。
   - **全局唯一最优**：所有有效承运商中运费最低者标记 isBest=1，恰 1 行；其余为 isBest=2。
5. 按 `isPushWms`、`pushResultTo` 推送至 SCP / WMS / ERP。

#### 计算规则

- **发运重**：取称重计算 `weight`，有小数向上取整；优先包裹层，无则取 SKU 层。**计费重取称重（实重），不取体积重；体积重仅落库展示，不参与计价**。本轮仅新增承运商要求金额精确可复算；已使用承运商沿用系统现有口径，测试只验证其能正常参与比价择优。
- **金额精度**：**每一步计算（单包裹运费、续重、燃油后金额、多包裹合计）均四舍五入保留三位小数**；中间结果不保留更高精度，最终金额同为三位小数。**币种暂不考虑**。
- **包裹的箱规取值计算**：支持多包裹计算，新增承运商只支持计算方式：一票一件；**qty>1 时按 qty 个相同 SKU 参与组包**
	- 若包裹层为0，则取sku的箱规信息；如果一个包裹存在多个sku，需要把sku的信息重新组包
		- 组包规则：
			- 取长边：对比所有 SKU，取最大的长度作为最终长边
			- 取宽边：对比所有 SKU，取最大的宽度作为最终宽边
			- 累加高：将所有 SKU 的高度进行累加求和
			- 累加重：将所有 SKU 的重量进行累加求和
			- 重排序：将最终得到的“长、宽、高”三个维度，按从大到小的顺序重新排列，拼接上总重量，得出最终箱规（前三步是计算过程，重排后的结果才是落库与计价使用的箱规）
- **箱规限制（不可发）**：最长边 >120cm或三边之和 >120cm或**原始称重（向上取整前）>22kg**，任一条件满足，即为不可发运；**箱规按单包裹逐个判定**。
- **总运费（税前）** = 基础运费 ×（1 + 燃油费率）
  - **fee 按重量段取值**：先按计费重匹配 `lsp_carrier_rate` 的 `min_weight < 计费重 ≤ max_weight` 段，取该段 `fee` 作为首重基础运费（同一承运商不同重量段 fee 不同，如 1.001~3.0 段 7.87、3.001~5.0 段 9.14）
  - 计费重 ≤5kg：基础运费 = `fee`（该段首重基础运费）
  - 计费重 >5kg：基础运费 = `fee + (计费重-5) × per_kilogram_fee`（Per KG 续重单价；fee 为 5.001+ 段 fee）
  - **附加费规则**（重要）：
    1. DB `lsp_carrier_rate.extra_fee` 列当前**不参与运费计算**（所有承运商价卡里都有该字段值，但系统未使用）
    2. 本轮新增承运商 AU.TGE.PRIORITY.PRO 无任何附加费
  - 燃油费率 = `lsp_carrier_rate.fuel_rate`（按承运商区分，**AU.TGE.PRIORITY.PRO 的 fuel_rate=0.2861**）
  - **zone 映射**：派件地址 postcode → zone 编码，查 `lsp_carrier_zone` 表；不同承运商 zone 划分不一样
- **计算方式说明**（**新增承运商 AU.TGE.PRIORITY.PRO 仅支持一票一件，响应中该承运商行 `calcMode` 恒为 1**）
	- 一票一件：拆单计算，每个包裹单独计算计费重量并匹配运费，最后将所有包裹的运费进行累加。
	- 一票多件：合单计算，将订单中的所有包裹视为一个整体。取所有包裹的最长边，累加所有包裹的计费重，生成一个合并后的“虚拟大包裹”，用这个整体数据去计算运费。
	 - 示例说明：
		- 包裹数据：
		- 包裹1：尺寸 30 12 10，实重 5kg（假设计费重为 5kg）
		- 包裹2：尺寸 60 20 15，实重 10kg（假设计费重为 10kg）
		- 一票一件（累加运费）：
		- 包裹1：用 30 12 10 + 计费重 5kg 代入公式 → 得出运费A
		- 包裹2：用 60 20 15 + 计费重 10kg 代入公式 → 得出运费B
		- 总运费 = 运费A + 运费B
		- 一票多件（合并计费）：
		- 提取最长边：对比两个包裹的最大长度，取 60
		- 累加总重量：5kg + 10kg = 15kg
		- 总运费 = 使用合并后的规格（最长边 60，总重量 15kg）代入公式计算 → 得出最终总运费
- **择优**：两层择优，见 §7 流程第 4 步。
  - **超时**：多个承运商参与计费时，任一承运商超时则整单报错（§6 500）。
  - **无价卡**：① 仅 1 个承运商参与计算且无价卡 → 计费失败，原因写 `lsp_quote_task_job_log.failed_reason`；② 多个承运商参与计算 → 无价卡承运商过滤，剩余承运商正常比价择优；③ 过滤后无任何可用承运商 → 计费失败。**无收件区域同本三分支口径。**
  - **多仓**：本接口暂不支持多仓，不涉及按仓拆分任务与按仓择优。
  - **超箱规**：若仅新增承运商参与且超箱规 → 计费失败；若还有其他承运商参与 → 过滤新增承运商，其他承运商正常计算比价。**单老承运商超箱规、`calcMode=2` 合并虚拟包裹的箱规判定不在本版本范围。**
  - **并列**：`shippingCost` 相同时，按 `data` 排序（`isBest=1` 优先，其余按 `shippingCost` 升序）取第一条为全局最优 `isBest=1`；排序稳定，同金额时按承运商计算顺序（策略/入参顺序）。
- **含税金额**（暂未启用）：shippingCost × 1.1；目前没有参与业务逻辑

### 7.2 AU.TGE.PRIORITY.PRO 计费规则（acceptance）

> 本节是本次新增承运商的验收口径。PRO 的价卡、公式、zone、fuel、箱规均为该承运商专属；7.1 中的通用框架若与本节的 PRO 参数冲突，以本节为准。

| 项 | PRO 口径 |
|---|---|
| 承运商 | AU.TGE.PRIORITY.PRO（`CAR2025062611067266788`） |
| calcMode | 仅支持一票一件，响应 `calcMode=1`；响应行 `canShipping` 恒为 true |
| 价卡匹配 | `from_zone`（发件仓）+ `to_zone`（派件 postcode 映射）+ `min_weight < 计费重 <= max_weight` |
| 首重/续重 | 计费重 ≤5kg：`fee`；>5kg：`fee + (计费重-5) × per_kilogram_fee` |
| 燃油 | `shippingCost = 基础运费 × (1 + fuel_rate)`；PRO `fuel_rate=0.2861` |
| 附加费 | 无附加费；`lsp_carrier_rate.extra_fee` 不参与计算 |
| 箱规 | 最长边 >120cm / 三边和 >120cm / 原始称重 >22kg 不可发；按单包裹判定 |
| 组包 | 包裹层全 0 时按 SKU 组包：长取 max、宽取 max、高累加、重累加、三维降序重排 |
| 精度 | 每一步（单包/续重/燃油/多包合计）四舍五入保留三位小数 |
| 过滤 | 多承运商场景：PRO 超箱规/无价卡/无收件区域时过滤 PRO，其余正常；仅 PRO 参与且失败时整单失败 |
| 主数据/承运商校验 | 未知仓库、VIR 中不存在的 carrierId → HTTP 500 + code="500"；未知仓库落 `status=300`，carrierId 不存在不落行 |
| 落库 | `lsp_quote_task`：`status=200`、`predict_price>0`、`predict_carrier_code` 非空、`predict_calc_mode=1`；地址 2 行、包裹/SKU 与请求一致、`job_log` 有 `action=calc` |

## 8. 非功能约束

| 项 | 值 | 说明 |
|---|---|---|
| 超时 | 30s | nacos 配置；**本接口以 30s 为准，不采用知识库中按承运商覆盖的 20s/15s** |
| 重试 | 不支持 | 单接口不重试；因不做幂等，调用方超时后不得自动重试，否则会重复生成计费任务 |
| 限流 | — | 不考虑 |
| 并发 | — | 不考虑 |
| 容量上限 | 包裹数：用例按 20 构造 | 业务暂未设上限；承运商数 / SKU 数上限未说明 |


## 9. 数据影响

### 9.1 准入校验（请求前读，决定能不能进入计费）

用于入参合法性校验和承运商筛选，不直接参与金额计算。

| 数据库表名 | 说明 | 测试用途 |
| :--- | :--- | :--- |
| `lsp_carrier` (承运商表) | 承运商基础配置表（启用状态等） | 断言承运商是否可参与计费 |
| `lsp_carrier_strategy` (策略表) | 承运商策略表 | 校验 `carrierStrategyCode` 是否存在 |
| `lsp_carrier_strategy_refs` (关联表) | 承运商策略关联的承运商 id 或者代表 | 断言策略下选中的承运商集合 |

### 9.2 计费输入（计算过程中读，直接影响金额）

计算时实时读取，**决定最终报价**。写用例时需要反算期望值。

| 数据库表名 | 说明 | 测试用途 |
| :--- | :--- | :--- |
| `lsp_carrier_zone` (收件地址表) | 收件地址 → 承运商 zone 的映射 | 确认收件 postcode 映射到正确的 zone，从而命中正确的价卡 |
| `lsp_address_suburb_postcode` (地址对照表) | 派件地址 suburb/postcode/state 校验，zone 反算链路起点 | zone 反算 SQL 起点（§4 说明表引用） |
| `lsp_carrier_rate` (价卡表) | 承运商价卡表（分段起点/终点、fee、per_kg、fuel） | 反算运费 |
| `lsp_warehouse` (仓库信息表) | 读取仓库主数据；`warehouseCode` 不存在时按**计费失败**处理（§9.3.2） | 断言失败任务落 `status=300` 行 |

### 9.3 计费结果（请求后写，用来断言是否成功）

请求结束后新写入或更新的表，是验证 SQL 的主要落点。以下表名与列名均以 `lsp_scp` 实际 schema 为准（2026-11-23 校准）。

| 数据库表名 | 说明 | 关键列（实查） |
| :--- | :--- | :--- |
| `lsp_quote_task` (计费结果表) | 计费结果及择优 | `request_id` / `order_code` / `warehouse_code` / `predict_price` / `predict_carrier_code` / `predict_calc_mode` / `status` / `close_pobox_check` / `is_po_box_addr` |
| `lsp_quote_task_address` (计费地址表) | 计费后发货地址和收件地址快照 | `quote_task_id` / `type`（100=发件 200=收件）/ `country` / `state` / `suburb` / `postcode` / `address1` / `address2` |
| `lsp_quote_task_package` (计费包裹表) | 计费包裹信息 | `quote_task_id` / `package_code` / `length` / `width` / `height` / `weight` / `cube` / `cube_weight` / `predict_price` / `status` |
| `lsp_quote_task_package_sku` (计费包裹的 SKU 表) | 包裹内 SKU 明细 | `quote_task_package_id`（→ package.id）/ `sku` / `qty` / `length` / `width` / `height` / `weight` / `cube` / `cube_weight` / `predict_price` |
| `lsp_quote_task_job_log` (计费日志表) | 参与计算的承运商及计费报错日志 | `quote_task_id` / `action`（calc / merge / choice）/ `carrier_codes` / `carrier_agent_code` / `failed_reason` / `status` / `calc_mode` / `rate_type` |

#### 9.3.1 成功路径落库契约（必断 / 可选快照分级）

`status` 取值：`200`=计费成功、`300`=计费失败、`100`=进行中。成功路径的断言分两级：

**必断（接口级用例必须校验）**

| 表 | 断言 |
| :--- | :--- |
| `lsp_quote_task` | **本接口单仓且不做幂等**：每次调用生成 1 行；**用例需保证 `requestId`/`orderCode` 唯一**，否则重复调用会产生多行。`status=200`；`predict_price>0`；`predict_carrier_code` 非空；`predict_calc_mode ∈ {1,2}` |
| `lsp_quote_task_address` | 该 `quote_task_id` 恰 2 行：`type=100`（发件，postcode=仓库 postcode）与 `type=200`（收件，postcode=派件 postcode）各 1 行 |
| `lsp_quote_task_package` | 请求包裹层有值时：行数 = 请求包裹数，尺寸/重量与请求值一致（三位小数精度）；请求包裹层字段全为 0 时由系统自动组包，行数与 `package_code` 以系统生成为准 |
| `lsp_quote_task_package_sku` | 经 `quote_task_package_id` 关联，行数 = 请求 SKU 明细行数；`sku` / `qty` 与请求一致 |
| `lsp_quote_task_job_log` | `action=calc` 至少 1 行；成功场景日志 `status=300` |

**可选快照（不校验，避免与系统生成规则冲突）**

- `package_code` 由系统生成；仅当请求包裹层有值时与请求 `packageCode` 一一对应，自动组包场景不保证；
- `cube` / `cube_weight` 由系统按箱规计算落库，不校验请求入参 `cube` / `cubeWeight`（见 §7：体积重仅落库展示，不参与计价）；
- `lsp_quote_task_job_log.result_data` 结构不校验；`canShipping=false` 的落库内容不做断言，用例只校验响应中该承运商行为一票一件。
	- cube计算公式：round(长*宽*高/1000000,3)
	- cube_weight计算公式：round(round（长*宽*高/1000000,3)*250,3)

#### 9.3.2 失败路径落库口径

| 失败类型 | `lsp_quote_task` 落库 | 用例断言口径 |
| :--- | :--- | :--- |
| 计算失败（无价卡/无收件区域且无可用承运商 / 未知仓库 / 找不到发运地址 / 超时等） | **落 1 行**：`status=300`、`predict_price=0`、`predict_carrier_code` 为空（2026-11-23 `lsp_scp` 实查 30 天 445 行 status=300 佐证） | 断言 `status=200` 的成功任务数为 0，**不得**断言任务总数=0 |
| 请求参数校验拒绝（缺必填：HTTP 200、code=400；空串/非法枚举等其余参数校验：HTTP 500、code=500；均进入计费前拦截） | 不落行 | 断言该 `request_id` 任务总数为 0 |
| 计费日志 | 失败任务同样写 `lsp_quote_task_job_log`：`status=400` + `failed_reason` 非空（实查现网有样例行） | 接口级用例对 `job_log` 只做**软断言**（查询并记录，不判失败），避免写入时机差异导致误报；具体失败类型是否必写日志待联调确认。 |

#### 9.3.3 测试数据清理

自动化用例按 `order_code LIKE 'TESTSO%'` 生成唯一业务键（`request_id` 前缀 `SAC` 仅作辅助识别；`requestId` 上界用例为 `R*64`，故清理只以 `order_code` 为准）。清理为人工/定时任务（脚本只读，不删数据）。按外键顺序执行：

```sql
DELETE s FROM lsp_quote_task_package_sku s
  JOIN lsp_quote_task_package p ON p.id = s.quote_task_package_id
  JOIN lsp_quote_task t ON t.id = p.quote_task_id
 WHERE t.order_code LIKE 'TESTSO%';
DELETE p FROM lsp_quote_task_package p
  JOIN lsp_quote_task t ON t.id = p.quote_task_id
 WHERE t.order_code LIKE 'TESTSO%';
DELETE l FROM lsp_quote_task_job_log l
  JOIN lsp_quote_task t ON t.id = l.quote_task_id
 WHERE t.order_code LIKE 'TESTSO%';
DELETE a FROM lsp_quote_task_address a
  JOIN lsp_quote_task t ON t.id = a.quote_task_id
 WHERE t.order_code LIKE 'TESTSO%';
DELETE FROM lsp_quote_task WHERE order_code LIKE 'TESTSO%';
```

同一份清理 SQL 已同步到 `config/lsp_quote/price_sync.yaml` 尾部注释，供运维直接取用。


## 10. 依赖影响

| 类型 | 内容 | 说明 |
|---|---|---|
| 下游服务 | 各承运商询价接口；SCP / WMS / ERP（推送） |  |
| 消息事件 | — | — |
| 副作用 | `isPushWms=100` 时可能写库或发送消息 |  |

## 11. 变更记录

| 版本 | 日期 | 变更点 | 影响范围 |
|---|---|---|---|
| v1.0 | 2026-11-18 | 新增承运商价卡计费：ubi-toll-AU.TGE.PRIORITY.PRO | 1. 新增承运商价卡计费<br>2. 比价择优加入新的承运商比价 |