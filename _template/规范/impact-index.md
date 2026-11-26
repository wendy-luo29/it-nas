# 影响索引（impact-index.yaml）规范

机读文件，技能与脚本直接解析以判断回归范围。

## 1. 结构

```yaml
changes:
  - id: CHG-001                 # 唯一，递增，不复用
    module: <模块>
    feature: <功能>
    field: <字段 / 变更单元>
    date: "yyyy-MM-dd"
    type: rule_change | api_change | db_change | pricing_change
    description: |
      变更内容与判定口径
    affected_carriers: [all] | [<承运商编码>]
    regression_targets: [TC-XXX-001, TC-XXX-002]
    coverage_gap: "<无自动化用例时说明缺口>"
    gotchas: [gotcha_001]
    knowledge_docs:
      - Common/业务规则/<module>/<feature>.md
    status: active | superseded
```

## 2. 维护规则

| 规则 | 说明 |
| --- | --- |
| 只增不改 | 每次变更新增条目，不覆盖历史 |
| ID 唯一 | `change_id` 全库唯一，`changelog` 必须能对应 |
| 回归目标真实 | `regression_targets` 的 case_id 必须同时存在于用例清单与脚本 |
| 缺口显式 | 确实没有自动化用例时置空并填写 `coverage_gap` |
| 文档可追溯 | `knowledge_docs` 路径必须存在 |
| 同提交更新 | 知识库改动与索引更新必须在同一次提交 |
| 废弃保留 | 历史条目改 `status: superseded`，不删除 |

## 3. 校验项

| 编号 | 校验内容 | 级别 |
| --- | --- | --- |
| B | 结构完整、ID 唯一、`knowledge_docs` 存在 | ERROR |
| C | `regression_targets` 的 case_id 存在（幽灵回归项） | WARN，`--strict` 下失败 |
| E | changelog 的 `change_id` 能在索引中找到 | ERROR |

## 4. 分仓场景

知识库与用例分仓时，用 `--knowledge-root` 与 `--testcases-root` 分别指向：
- 知识库根（`<Project>/`，即包含 `Common/` 与 `impact-index.yaml` 的目录）
- 用例清单（代码仓 `testcases/`）

`knowledge_docs` 的路径相对知识库根解析；指向代码仓文件（如 `config/...`）时回退按代码仓根解析。`regression_targets` 的解析基准是 `--testcases-root`，跨仓引用需在 CI 中同时 checkout 两个仓库。
