# gotcha_002：DB 字段 NOT NULL 但 API 返回空串

## 踩到什么坑
`data.address` 字段在 DB 层有 NOT NULL 约束，但接口文档没说 API 层是否强制非空。
实际行为：address 传 null 时，API 会存空串到 DB。

这导致用例断言 `assert data["address"] is not None` 和 `assert data["address"] != ""` 的边界不确定。

## 正确做法
- 先读 OpenAPI schema + DB information_schema 确认字段约束
- 区分三层约束：DB 层（NOT NULL）/ API 层（OpenAPI required）/ 业务层（规则文档）
- 测试断言应反映实际确认过的行为，不要假设
- 不确定时在 `Common/业务规则/` 标【待确认】并提给业务方

## 关联
- 工具：`scripts/query_db_schema.py` 可自动查询 DB 约束
