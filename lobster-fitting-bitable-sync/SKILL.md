---
name: lobster-fitting-bitable-sync
description: 将试衣 input-pipeline 结果映射成适合飞书多维表格写入的结构。适用于需要把试衣记录稳定写入飞书主表，并对必填字段做强制校验的场景。v2 相对 v1：字段集对齐 v2 contract，校验委托给 lobster-input-validator，binding 一致性按 binding-config-protocol。
---

# 试衣飞书多维表格沉淀（v2）

## ⚠️ 写入前最高优先级护栏（小模型必读）

**接到 pipeline_result 后，第一件事不是写表，而是检查以下闸门**。任一项失败 → `sync_status = blocked`，立即返回 `required_field_actions`，**绝不进入 +record-create / +record-update**。对 MiniMax 等小模型尤其关键：不要因为字段都有占位值就以为可以写。

| 闸门 | 失败 → 立即返回 |
|---|---|
| `pipeline_result.write_decision.allow_write == true` | 否则 `sync_status = blocked`，原因取自 `write_block_reason` |
| `pipeline_result.input_bundle.bundle_status != pending_media` | 否则 `sync_status = blocked: PENDING_MEDIA_NOT_READY_TO_WRITE` |
| `guide_name` 非空且不在 `["导购","店员","销售","营业员","未知","默认导购","测试导购","N/A",""]`，且不以 `ou_` 开头 | 否则返回 `ask_user: 请补充导购姓名` |
| `store_name` 非空 | 否则返回 `ask_user: 请补充门店名称` |
| `operator_id` 以 `ou_` 开头 | 否则 `sync_status = blocked`，connector 必须重新透传 |
| `fitting_at` 来自 message_metadata.timestamp / 导购显式提供 / 卡片确认（不得为模型编造） | 否则 `sync_status = blocked: FITTING_AT_FABRICATED` |
| `product_code` / `product_name` / `raw_notes` 非空 | 否则 `sync_status = blocked: MISSING_REQUIRED_FIELDS` |
| 调用 `lobster-input-validator` 后 `can_write == true` | 否则 `sync_status = blocked`，原因从 validator_result 透传 |

**强约束**：上述任一闸门未通过时，本 skill **不得**执行 `+record-create` / `+record-update`；也不得为了"先把能写的写进去"而部分落表。

权威规则见 [`../references/schema.json`](../references/schema.json) 的 `write_gate_rules`。

---

把结构化试衣数据整理成适合飞书多维表格写入的结果。

## 依赖

- 当任务涉及飞书多维表格字段设计、记录写入、视图或公式语义时，优先配合 `lark-base` 类能力执行
- 若当前缺少 `bitable_binding`、Base 或目标表结构，先使用 `lobster-fitting-bitable-bootstrap` 补齐基础设施，再生成正式写表载荷
- 写表前必须先读取 [`../references/bitable-field-contract.md`](../references/bitable-field-contract.md)，按 contract 判断字段语义、最小可写字段集和可疑字段
- v2 中字段校验委托给 `lobster-input-validator`，本 skill 不重复实现校验逻辑

## 输入

- `pipeline_result`：v2 中由 `lobster-fitting-input-pipeline` 提供
- `fitting_records`
- `media_records`
- `write_decision`
- `bitable_binding`（按 v2 binding-config-protocol 包含 `field_map` / `last_verified_at`）
- `instance_context`（可选；用于补 `store_name / operator_id` 等来源字段）
- `field_sources`（建议；由 input-pipeline / card-fill 透传，用于校验 required 字段来源可信度）

## binding 校验流程

写入前按 [`binding-config-protocol`](../references/binding-config-protocol.md) 校验 binding：
1. 检查 `binding.binding_status == ready`
2. 检查 `binding.last_verified_at` 距今 < 24h
   - 超时 → 触发轻量字段校验（调用 list-fields 比对 `field_map`）
   - 不一致 → 触发 bootstrap 修复
3. 校验通过 → 进入写表

## 字段检查流程

写表前不要直接落表，先做字段检查：
1. 调用 `lobster-input-validator(target_table=试衣商品记录, payload, binding_field_map)`
2. 拿到 `can_write` / `missing_required_fields` / `enum_violations` / `suspicious_fields`
3. 只有 `can_write = true` 时，才允许进入正式写入
4. 发现 `enum_violations` 或 `suspicious_fields` 时，应优先提示治理，不要继续混写
5. 若 `guide_name` 缺失或可疑，必须返回追问动作，不得用 `导购`、`店员`、`默认导购` 等值补齐后写入

## 主表字段映射（v2）

### fitting_record → 试衣商品记录

v2.1 主表字段集（20 字段）：
- `session_id`
- `guide_name`
- `fitting_at`
- `store_name`
- `operator_id`
- `member_mobile_last4`
- `product_code`
- `product_name`
- `color`
- `size`
- `fabric`
- `category`
- `tag_price`
- `body_effect_desc`
- `fit_feedback`
- `liked_points`
- `disliked_points`
- `not_buy_reason`
- `followup_intent`
- `raw_notes`
- `media_urls`

### required 字段（v2.1，7 个）
- `session_id`
- `guide_name`
- `fitting_at`
- `operator_id`
- `product_code`
- `product_name`
- `raw_notes`

若任一字段缺失：
- `write_mode` 不得进入正式写入
- `sync_status` 应标记为 `partial` 或 `blocked`
- `sync_summary` 中必须明确列出缺失字段

### 关键写入规则

- `session_id` 是主表唯一标识；不再依赖 `record_id`
- `operator_id` 必须直接透传飞书 sender open_id，不依赖模型提取（v1 的 `source_bundle_id` 已删除，统一由 `operator_id` 承担）
- `guide_name` 必须来自本次明确自报、已确认 profile、卡片确认或可信员工资料；缺失时阻断并追问，不得写角色名/占位值
- `store_name` 缺失时同样必须追问，不得静默写入空值
- `member_mobile_last4` 非空 → 视为会员（v1 的 `is_member` 字段已删除）
- `product_name / product_code` 必须优先取本次识别结果，不能被历史上下文商品覆盖
- `fitting_at` 是 datetime 单字段：
  - 写入值（业务时间）统一格式 `YYYY-MM-DD HH:mm:ss`
  - 字段在飞书侧 `date_formatter` 必须为 `yyyy/MM/dd HH:mm`（含时间分量）；若发现字段格式只是 `yyyy/MM/dd`，应触发 bootstrap 修复，**不要继续写**
  - 来源必须为 `message_metadata.timestamp` / 导购显式提供 / 卡片确认；模型不得编造日期

## 反馈四字段写入

主表的 `body_effect_desc` / `fit_feedback` / `liked_points` / `disliked_points` 由上游 AI 主动拆分填充：
- 能拆出哪个就写哪个
- 不强求每次写满；**无明确语义匹配的字段必须留空**，不要塞中性描述
- 不允许把多个字段合并到单字段

**写入前可疑值检测**（针对 MiniMax 等小模型的常见错误模式）：
- `liked_points` 命中以下模式必须丢弃（置为空）：`*试穿*`（无其他正面词）/ `*拿了一件*` / `*看了一下*` / `*试了一下*` / 与 raw_notes 全文相同
- 若 raw_notes 含 `不X` / `X不Y` / `有点X` 句式但 `disliked_points` 和 `fit_feedback` 同时为空，应标记 `merge_notes` 警告"可能漏抽负面反馈"
- 同一负面点（如"不贴合"）可同时出现在 `fit_feedback` 和 `disliked_points`，不视为重复

本 skill 写入时，按字段独立写入；不做合并加工，但要按上述模式做最后一次过滤。

## dedupe 建议

主表建议使用：
- `session_id`

## 输出目标

输出 `sync_result`：
- `tables`
- `write_mode`
- `dedupe_keys`
- `sync_status`
- `sync_summary`
- `validator_result`（v2 新增；来自 lobster-input-validator）

## 执行规则

- 缺 binding 时，不要伪造固定表配置；应先进入"人工给链接 → 自动登记 binding"流程
- 若既没有 binding，也没有可用链接，`sync_status` 应标记为 `needs_confirmation` 或 `no_binding`
- 若 `validator_result.can_write = false`，应阻断正式写入
- 若 `validator_result.suspicious_fields` 包含 `guide_name`，应将 `sync_status` 置为 `blocked`，并在 `sync_summary` 中明确要求补充导购姓名
- 若只拿到 `session` 级摘要、尚未拆成可写主记录，不要擅自落表
- 若 `write_decision.allow_write = false`，连接器必须停止向主表写入
- 若 `pipeline_result.input_bundle.bundle_status = pending_media` 且 `fitting_records = []`，不得向主表写入任何正式记录
- v2 强约束：写入前必须调用 `lobster-input-validator`，不再在本 skill 内重复校验逻辑

## 输出格式

```json
{
  "sync_result": {
    "tables": {
      "试衣商品记录": []
    },
    "write_mode": "upsert | blocked",
    "dedupe_keys": {
      "试衣商品记录": ["session_id"]
    },
    "sync_status": "ready | partial | blocked | pending_confirm | needs_confirmation | no_binding",
    "sync_summary": "...",
    "validator_result": {
      "can_write": true,
      "missing_required_fields": [],
      "enum_violations": [],
      "suspicious_fields": []
    }
  }
}
```
