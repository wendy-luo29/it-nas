# IT NAS 测试知识库

按 `项目 / Common / UR 批次` 组织的测试知识库：需求证据、业务规则、踩坑、变更记录、影响索引、用例与报告都沉淀在这里；代码仓只放可执行脚本，并以 submodule 挂载本库读取知识。

## 目录规范

```text
it-nas/
├── README.md                       # 本文件：知识库规范与校验入口
├── _template/                      # 新项目 / 新需求批次的复制模板
│   ├── 规范/                       # 知识库规范：目录分层、规则 front-matter、影响索引
│   ├── rule.md / gotcha.md / changelog.md / impact-index.yaml
│   ├── 项目说明.md
│   └── UR-xxx-yyyyMMdd/
├── tools/
│   └── knowledge_guard.py          # 一致性校验器（CI 必跑）
└── <ProjectName>/                  # 一个产品或项目线一个目录
    ├── 项目说明.md                  # 项目简介、模块说明、历史需求索引
    ├── impact-index.yaml           # 机读影响索引：变更单元 → 回归目标用例
    ├── Common/                     # 跨需求复用的稳定结论
    │   ├── 知识库索引.md            # AI 必读入口：按问题导航
    │   ├── 用户使用手册.md
    │   ├── 架构设计文档.md
    │   ├── 业务规则/<module>/<feature>.md
    │   ├── 踩坑记录/gotcha_NNN_<主题>.md
    │   ├── 变更记录/yyyy-MM-dd_<主题>.md
    │   └── 参考/                    # 术语、关键表结构、接口文档入口
    └── UR-xxx-yyyyMMdd/            # 需求批次，自包含，可独立交接
```

每个 `UR-xxx-yyyyMMdd/` 为一个需求批次：

```text
UR-xxx-yyyyMMdd/
├── 版本说明.md      # 版本概况：需求描述、功能点、改动点、分支、接口、数据库
├── 用户需求/        # 需求原文与评审确认后的范围说明
├── 研发分析/        # AC 拆解、技术分析、评审记录、边界矩阵
├── 接口文档/        # OpenAPI / JSON 契约快照
├── 数据库文档/      # 本版本涉及的库表、字段、主数据
├── 发布文档/        # 代码分支、基线 commit、变更 commit、回滚方案
├── 测试报告.md      # 执行结论、覆盖范围、缺陷映射、跳过说明
└── 测试用例/
    ├── 用例文件/    # Excel / YAML / JSON 用例清单
    └── 脚本文件/    # 自动化脚本索引（指向代码仓路径与 commit，不复制源码）
```

## 两层分离

| 层 | 位置 | 可变性 | 内容 |
| --- | --- | --- | --- |
| 证据层 | `<Project>/UR-*/`（用户需求、接口文档、数据库文档、研发分析、发布文档） | 不可变快照 | 需求原文、契约、评审记录、AC 清单 |
| 结论层 | `<Project>/Common/` + `<Project>/impact-index.yaml` | 可维护，需 front-matter | 业务规则、踩坑、变更记录、参考、影响索引 |

禁止把需求原文抄进结论层；结论层只放经确认的判断和来源引用。

## 与代码仓的接入方式

代码仓通过 submodule 挂载本库，知识库根 = `knowledge/<ProjectName>/`：

```powershell
git submodule add <本库地址> knowledge
git submodule update --init --recursive
```

Skill 读取知识库时按「知识库根」相对路径定位，不硬编码代码仓目录。

## 校验

```powershell
# 在代码仓内（知识库挂在 knowledge/）
python knowledge/tools/knowledge_guard.py `
  --project-root . `
  --knowledge-root knowledge/Project-LspQuote `
  --strict

# 直接在本库内
python tools/knowledge_guard.py --knowledge-root Project-LspQuote --strict
```

退出码 0 且输出「0 个错误，0 个警告」才算通过。

## 命名规范

| 对象 | 规则 | 示例 |
| --- | --- | --- |
| 项目目录 | 大驼峰，与产品线一致 | `Project-LspQuote` |
| 需求批次目录 | `UR-<需求编号>-<首次评审日期>` | `UR-002-20261120` |
| 需求编号 | 沿用业务侧 REQ 编号，不改写 | `REQ-002` |
| 规则文件 | `业务规则/<module>/<feature>.md`，文件名不带版本 | `业务规则/lsp_backend/address_validation.md` |
| 踩坑文件 | `踩坑记录/gotcha_NNN_<主题>.md`，编号只增不改 | `踩坑记录/gotcha_001_pobox_case.md` |
| 变更记录 | `变更记录/yyyy-MM-dd_<主题>.md` | `变更记录/2026-11-20_quote_carrier_a_v2.md` |
| 版本说明 / 测试报告 | 固定文件名 | `版本说明.md` / `测试报告.md` |

> 目录用 `UR-` 前缀，正文与用例 `case_id` 继续使用 `REQ-` 前缀；两者等价关系在每个 `版本说明.md` 的「标识对照」一节声明。

## 新增需求批次的操作步骤

1. 复制 `_template/UR-xxx-yyyyMMdd/` 到目标项目目录下，重命名为 `UR-<编号>-<日期>`。
2. 填入 `版本说明.md`，确认分支、接口、数据库三项与研发一致。
3. 需求、分析、接口、数据库文档按目录归位。
4. 用例文件落 `测试用例/用例文件/`，脚本在代码仓开发，`测试用例/脚本文件/脚本索引.md` 只登记路径与 commit。
5. 执行完成后再落 `测试报告.md`，缺陷单随报告一起归档。
6. 在项目 `项目说明.md` 的历史需求索引表追加一行。
7. 评审确认的规则升级到 `Common/业务规则/`，同步新增 `Common/变更记录/` 与 `impact-index.yaml` 条目，再跑一次校验。

## 归档要求

- `发布文档/` 必须记录代码分支与 commit，保证 AI 或他人能通过 git 日志反查当次改动代码。
- 接口文档与数据库文档以本版本实际生效的契约为准，不写"待补充"。
- 测试报告必须包含执行摘要、覆盖范围、缺陷映射、跳过原因，不允许只放一行通过数。
- 规则、踩坑、变更记录与影响索引必须同时提交，`knowledge_guard.py --strict` 无告警才可归档。
