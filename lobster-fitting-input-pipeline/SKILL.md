---
name: lobster-fitting-input-pipeline
description: 把飞书会话中的图、音、文消息一站式归并、做 session 判断、合并多源字段、判定 pending_media。适用于试衣消息进入草稿前的统一前置流水线。本 skill 合并了 v1 的 lobster-fitting-bundle-router 与 lobster-fitting-record-merge。
---

# 试衣输入流水线（v2 合并 skill）

## 目标
把 v1 中"先 bundle-router 归并、再 record-merge 合并字段"两步合并为一次调用，对外只暴露一个 skill，但内部仍按阶段执行，保留 `pending_media` 状态机。

## 内部阶段

```
阶段1：消息归并（原 bundle-router 职责）
  ↓
阶段2：session 判断与关联
  ↓
阶段3：多源字段合并（原 record-merge 职责）
  ↓
阶段4：pending_media 与 write_decision 判定
```

## 输入来源
- `messages`：当前要处理的消息列表（已包含 transcript 注入后的语音）
- `sender_id` / `chat_id`
- `message_metadata`（建议）：至少包含飞书 sender open_id、消息时间，用于透传 `operator_id` 和生成 `fitting_at`
- `existing_open_bundle`（可选）
- `existing_open_draft`（可选）
- `context_policy`（可选，`inherit_if_safe` / `strict_isolation`）
- `reset_context`（可选，`true` 时表示当前消息应与旧 bundle 隔离）
- `force_new_bundle`（可选）
- 上游 skill 输出：
  - 导购原始文字
  - 语音转写结果（已注入 messages[].text 或单独传入 `voice_extract_result`）
  - 吊牌识别结果 `ocr_result`
  - 会员识别结果 `member_result`
- 上下文：`sender_profile` / `guide_profile` / 最近 open session

## 输出目标
返回 `result`：
- `input_bundle`
- `session_link`
- `fitting_records`
- `media_records`
- `write_decision`
- `merge_notes`
- `missing_fields`
- `required_field_actions`

### input_bundle
- `bundle_id`
- `bundle_status` (`pending_media` / `single` / `merged`)
- `message_ids`
- `message_types`
- `window_seconds`
- `routing_action` (`append_bundle` / `new_bundle` / `keep_waiting`)
- `routing_reason`

### fitting_records
每项尽量包含主表保留字段（v2.1 字段集，20 字段）：
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

### write_decision
- `allow_write`
- `write_block_reason`
- `write_target`

## 阶段1：消息归并规则
- 默认归并窗口：`120s`
- 同一导购、同一会话、在时间窗口内的图 / 音 / 文优先归入同一个 `bundle`
- 单张吊牌图先进入 `pending_media`
- 若窗口内补充了语音或文本，则 `bundle_status` 更新为 `merged`
- 若超时无补充，保留 `pending_media` 并交给 draft-manager 决定继续等待或提醒补充
- 不得仅因"最近一条记录存在"就自动把新图片归到历史正式记录
- 若 `context_policy = strict_isolation`，应默认降低 `existing_open_bundle` 的续接优先级
- 若 `reset_context = true` 或 `force_new_bundle = true`，必须输出 `routing_action = new_bundle`
- 若当前消息与 `existing_open_bundle` 跨自然日，即使仍处于 open 状态，也不得续接

## 阶段3：多源字段合并规则

### A. 必填字段前置校验
要进入正式写表候选，以下字段必须可被稳定提取（v2.1 主表 7 个 required）：
- `session_id`
- `guide_name`
- `fitting_at`
- `operator_id`
- `product_code`
- `product_name`
- `raw_notes`

若任一字段缺失：
- 必须写入 `missing_fields`
- `write_decision.allow_write` 必须为 `false`
- `write_block_reason` 应明确为 `MISSING_REQUIRED_FIELDS`

### B. 导购资料继承规则
- 若当前消息未再次显式提供 `guide_name` / `store_name`，但 `sender_profile` 中已有稳定值，直接继承
- `operator_id` 应优先直接取自入站消息 metadata（飞书 sender open_id），不由模型从正文推断
- 不得因为单条消息缺少姓名/门店，就忽略同一 sender 在历史中已经明确给过的信息
- 只有在 `sender_profile` 不存在、或当前输入与历史资料冲突时，才允许将其标记到 `missing_fields`
- `guide_name` 不得从"说话人是导购"这一角色事实推断为 `导购`、`店员`、`销售`、`默认导购` 等占位词
- 当 `guide_name` 缺失且 `sender_profile/guide_profile` 没有已确认展示名时，必须把 `guide_name` 写入 `missing_fields`，并在 `required_field_actions` 中输出追问动作
- 若本次文本出现"我是导购/我是店员"但没有姓名，这不构成可写的 `guide_name`

### C. 商品主值约束
- `product_code` / `product_name` 应优先采用本次 bundle 的 OCR 结果 + 本次文本补充
- 若 OCR 结果与历史上下文商品冲突，应在 `merge_notes` 中记录冲突，并优先保留本次输入商品
- `fabric` / `category` 优先从 OCR 提取；如 OCR 未识别，可从文本/语音补充

### D. 反馈四字段填充
基于导购原始输入主动做语义拆分，分别填充：
- `body_effect_desc`：客户实际穿着后体型呈现
- `fit_feedback`：版型反馈
- `liked_points`：客户主观喜欢点
- `disliked_points`：客户主观不喜欢点

不强求每次写满四个；能拆出哪个写哪个；不要合并到单字段。

### E. 编造限制
- 不得编造必填字段
- 在只有图片、没有明确文本反馈时，不得编造成交、试穿结果、是否购买、跟进意向等业务结论
- 若无法稳定拿到必填字段，应阻断写入，而不是部分落表
- 禁止通过通用角色名、占位词或默认值补齐 required 字段；这类值应进入 `suspicious_fields` 或 `missing_fields`

## 阶段4：pending_media 与 write_decision 判定

强约束（与 v1 保持一致）：
- `bundle_status = pending_media` 且缺少可用文本描述时：
  - `fitting_records = []`
  - `write_decision.allow_write = false`
  - `write_block_reason = PENDING_MEDIA_NOT_READY_TO_WRITE`
- 任一 required 字段缺失：
  - `write_decision.allow_write = false`
  - `write_block_reason = MISSING_REQUIRED_FIELDS`
- 历史商品与当前 OCR 冲突：
  - `merge_notes` 记录冲突
  - 不阻断写入（只要本次商品稳定）

## 输出格式
```json
{
  "result": {
    "input_bundle": {
      "bundle_id": "...",
      "bundle_status": "pending_media | single | merged",
      "routing_action": "append_bundle | new_bundle | keep_waiting"
    },
    "session_link": {},
    "fitting_records": [],
    "media_records": [],
    "write_decision": {
      "allow_write": false,
      "write_block_reason": "PENDING_MEDIA_NOT_READY_TO_WRITE",
      "write_target": "仅缓存等待"
    },
    "merge_notes": [],
    "missing_fields": [],
    "required_field_actions": [
      {
        "field": "guide_name",
        "action": "ask_user",
        "message": "请补充导购姓名，之后同一导购可自动沿用。"
      }
    ]
  }
}
```
