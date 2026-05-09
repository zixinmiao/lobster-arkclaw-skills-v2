---
name: lobster-fitting-bitable-bootstrap
description: 在试衣数据准备写入飞书多维表格前，检查并补齐 bitable_binding、Base、数据表和必要字段，并按 v2 binding-config-protocol 持久化 binding。v2.1 改为 schema-driven：直接读取 references/schema.json 作为权威字段定义，建表/补字段后回读校验，不再凭文档自然语言重新解读。
---

# 试衣飞书表格初始化与绑定修复（v2.1，schema-driven）

## 目标
在正式写入前，先完成飞书表链接初始化或基础设施补齐，并输出稳定可复用的 `bitable_binding`，按 [`binding-config-protocol`](../references/binding-config-protocol.md) 双层持久化。

**v2.1 核心改进**：本 skill 改为 schema-driven。建表/建字段必须读取 [`../references/schema.json`](../references/schema.json) 作为机器可读权威定义；所有 select 字段的选项值、required 字段的列表、字段类型映射，都来自 schema.json，不再凭 markdown 自然语言重新解读。这是为了根治"两个 agent 对同一个 SKILL.md 解读出不同字段集"的问题。

## 何时使用
- 首次部署导购小龙虾，尚未创建试衣 Base
- 上游没有 `bitable_binding`
- 已有 `base_token`，但缺少目标表或必要字段
- 查询 / 写入返回 `no_binding`、`missing_binding`、`invalid_binding`
- `last_verified_at` 超过阈值（默认 24h）需要重新校验

## 推荐初始化方式

优先使用"人工给链接，首次自动登记 binding"：
1. 人工提供飞书 Base / Table 链接
2. 从链接中直接解析 `base_token / table_id / view_id`
3. 调用飞书读取真实字段结构，验证目标表是否可用
4. 构建完整 `field_map`（表 → 字段名 → field_id）
5. 将结果按 binding-config-protocol **双写**到平台 KV + `_系统配置` 表
6. 后续所有 open claw 直接复用这份 binding

## 输入
- `bitable_binding`（可选）
- `base_name`（可选，默认 `导购小龙虾试衣台账`）
- `folder_token`（可选）
- `time_zone`（可选，默认 `Asia/Shanghai`）
- `schema_version`（默认 `v2`）
- `expected_tables`（可选）
- `binding_scope`（默认 `project`）
- `provided_link`（可选；用户提供的 Base / Table 链接）

## 默认对象

### Base
- `导购小龙虾试衣台账`

### 必要表（v2，7 张）
- `试衣商品记录`（主表）
- `试衣Session索引`
- `试衣素材索引`
- `线索回访表`
- `会员画像表`
- `提醒派发表`（v2 从回访表分离）
- `_系统配置`（v2 新增，binding 自举入口）

## 工作流

### 1. 先检查 binding 或初始化链接
- 默认使用 **项目级共享 binding**：`binding_scope = project`
- 优先读 [`binding-config-protocol`](../references/binding-config-protocol.md) 定义的两层兜底
  - 第一层：平台 KV `binding:${project_id}`
  - 第二层：飞书 Base 内 `_系统配置` 表
- 若系统中已存在可用项目级 binding，则后续所有导购直接复用该 Base
- 若用户已提供 Base / Table 链接，优先直接解析链接并初始化 binding
- 不按 `guide_name`、`operator_id`、`sender_id`、`session_id` 创建新的 Base
- 只有在"没有 binding、也没有可用链接"时，才进入建表或 fallback 搜索流程
- 若输入是 wiki 链接，先解析真实 `base_token`，不要直接把 wiki token 当 Base token
- 若发现同一 `project_id` 下已有多个候选 Base，不得继续新建第三个；必须返回 `duplicate_binding_candidates`，要求人工指定唯一 canonical Base 后再写入

### 2. 先尝试通过链接直达并校验
- 若已拿到链接，先解析 `base_token / table_id / view_id`
- 直接读取真实表结构，确认该表可访问且字段可检查
- 访问成功后立即记录 `base_token`、`table_id`、`view_id`、`base_url`
- 只有链接不可用或表不存在时，才退回到建表/补表流程
- 不为"可能已有但未确认"的情况重复创建多个 Base

### 3. 若需建表，schema-driven 串行执行
**先读取 [`../references/schema.json`](../references/schema.json)**，按 `tables[]` 数组遍历：
- 对每张表 `t`，调用 `+table-list` 检查 `t.table_name` 是否存在
  - 存在 → 复用
  - 不存在 → `+table-create --table-name "{t.table_name}"`
- 7 张表（含 `_系统配置`）必须全部存在
- 所有 list / create 操作串行执行，避免并发冲突
- 不得只建主表跳过其他 6 张

### 4. 校验字段并补齐（schema-driven）
对 schema.json 中每张表的每个字段 `f`：
- `+field-list` 读取线上真实字段
- 比对：
  - 字段名缺失 → `+field-create --json '{"field_name":"{f.name}", "type":"{type_id}", ...}'`，**字段名严格使用 `f.name`，不得做大小写/翻译变体**
  - 字段类型不一致 → `+field-update`（注意飞书侧某些类型变更受限，必要时报错而非擅自改）
  - select 字段必须按 `f.options` 预置选项；不允许字段建好后让用户填
- 字段名映射规则：
  - schema 中 `type=text` → 飞书 Text（短文本/长文本由 description 暗示，统一用 Text）
  - schema 中 `type=datetime` → 飞书 DateTime（**严禁建成 Text**）
  - schema 中 `type=select` → 飞书 SingleSelect + 预置 options
  - schema 中 `type=number` → 飞书 Number
  - schema 中 `type=checkbox` → 飞书 Checkbox
  - schema 中 `type=url` → 飞书 Url 或 Text（飞书 Text 字段也能存 URL）
- 字段名禁止变体（典型）：`guidename` / `guideName` / `导购` / `导购姓名` / `导购名称` 不得替代 `guide_name`；完整禁止表见 schema.json 的 `field_naming_rules.forbidden_synonyms`
- 若线上已有同义/变体字段，不要把写入映射到变体字段，应补齐 canonical 字段并在 `schema_status` 中标记 `drift_detected`，同时 log 出变体字段名供人工治理

### 4.1 回读校验（v2.1 关键，治本）
建表+建字段后，**必须再次 `+field-list`** 比对 schema.json：
- 对 schema 中每个 required 字段，检查线上是否存在且类型一致
- 缺失 → `schema_status = drift_detected`，binding_status 不得置为 `ready`
- 类型不一致（例如 `fitting_at` 建成了 text）→ `schema_status = type_mismatch`，binding_status 不得置为 `ready`
- 出现 schema 中没有的字段（如"导购小龙虾试衣台账"误成字段）→ `schema_status = unexpected_fields`，log 出来供人工删除
- 只有上述检查全通过，才进入第 5 步构建 `field_map`

### 5. 构建 field_map
- 读取每张表的字段列表
- 把表 → 字段名 → field_id 映射写入 `binding.field_map`
- 这一步是 v2 关键改动，避免下游写入前每次重新读字段

### 6. 双层持久化（v2 关键）
按 [`binding-config-protocol`](../references/binding-config-protocol.md) 执行：
1. 写入平台 KV：`KV.put(binding:${project_id}, binding_json)`
2. 写入 `_系统配置` 表：`upsert(config_key=binding:project, config_value=JSON.stringify(binding))`
3. 设置 `last_verified_at` 为当前时间
4. 设置 `binding_status = ready`

任一持久化失败，应明确返回失败原因，不得假装成功。

### 7. 输出稳定 binding

返回结果至少包含：
- `base_token`
- `base_url`
- `tables`（表名 → table_id）
- `field_map`（表 → 字段名 → field_id；v2 关键）
- `binding_status`
- `schema_status`
- `last_verified_at`
- `bootstrap_summary`

## 核心规则

- 建表与写表分离：本 skill 只负责"确保可写"，不负责写业务记录
- 幂等优先：重复执行时，应补齐缺口，不应制造重复 Base / 重复表
- 默认使用项目级共享 Base：一个项目一套 Base，多导购共用
- 结构优先：先确保 Base、表、字段，再进入 `lobster-fitting-bitable-sync`
- 初始化检查范围包含全部 7 张表（含 `_系统配置`）
- bootstrap 阶段应以 required 字段可写为优先目标，而不是追求字段越多越好
- binding 缺失时，不要只返回 `no_binding` 就结束；应优先尝试"用户给链接 → 自动登记 binding"，再考虑补齐基础设施
- 若已存在可用 binding，则禁止因新导购触发而再次创建 Base
- 若没有可用的项目级持久化能力（KV 与 `_系统配置` 均不可写），不得返回 `binding_status = ready`；应返回 `partial` 或 `needs_confirmation`，否则不同 agent 会各自建表
- 若任一步失败，明确返回失败原因，不伪造 `ready`
- **v2 强约束**：binding 必须双写（KV + `_系统配置`），单写视为 partial

## 与其他 skill 的关系

- 上游可来自 `lobster-fitting-input-pipeline`、`lobster-fitting-draft-manager`、`lobster-fitting-card-fill`
- 建表完成后，再交给 `lobster-fitting-bitable-sync` 生成写表载荷
- 真实建表、建字段、查表结构时，优先配合 `lark-base` 执行
- 校验委托给 `lobster-input-validator`

## 输出格式

```json
{
  "bitable_binding": {
    "schema_version": "v2",
    "binding_scope": "project",
    "project_id": "lobster-fitting-shop-001",
    "base_token": "bascn_xxx",
    "base_url": "https://xxx.feishu.cn/base/bascn_xxx",
    "tables": {
      "试衣商品记录": "tbl_xxx",
      "试衣Session索引": "tbl_xxx",
      "试衣素材索引": "tbl_xxx",
      "线索回访表": "tbl_xxx",
      "会员画像表": "tbl_xxx",
      "提醒派发表": "tbl_xxx",
      "_系统配置": "tbl_xxx"
    },
    "field_map": {
      "试衣商品记录": {"session_id": "fld_xxx", "...": "..."}
    },
    "created_at": "2026-05-09T10:00:00+08:00",
    "last_verified_at": "2026-05-09T10:00:00+08:00",
    "binding_status": "ready",
    "schema_status": "ready"
  },
  "bootstrap_summary": "...",
  "next_action": "go_sync | retry_fix | manual_check"
}
```
