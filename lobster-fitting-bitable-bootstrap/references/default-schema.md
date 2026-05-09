# 默认建表结构（v2）

在执行建表或补字段前，先按真实表结构核对；以下是 v2 默认最小 schema。

## 1. 试衣商品记录（主表）

用于沉淀单客单商品单次试衣主记录。

建议字段（v2，21 字段）：
- `session_id`（文本）
- `guide_name`（文本）
- `fitting_at`（日期时间）
- `store_name`（文本）
- `operator_id`（文本，导购飞书 open_id）
- `member_mobile_last4`（文本）
- `product_code`（文本）
- `product_name`（文本）
- `color`（文本）
- `size`（文本）
- `fabric`（文本，v2 新增）
- `category`（单选；选项：上装 / 下装 / 连衣裙 / 外套 / 配饰）
- `tag_price`（数字）
- `try_on_result`（单选；选项：合适 / 待考虑 / 未成交 / 已购）
- `body_effect_desc`（长文本）
- `fit_feedback`（长文本）
- `liked_points`（长文本）
- `disliked_points`（长文本）
- `not_buy_reason`（长文本）
- `followup_intent`（文本）
- `raw_notes`（长文本）
- `media_urls`（长文本，多 URL 用换行分隔；或飞书"附件"字段）

建议去重键：
- `session_id`

必填校验字段（v2，8 个）：
- `session_id`
- `guide_name`
- `fitting_at`
- `operator_id`
- `product_code`
- `product_name`
- `try_on_result`
- `raw_notes`

## 2. 试衣Session索引

用于追踪 session 生命周期和 bundle 归属。

建议字段：
- `session_id`（文本）
- `link_action`（文本）
- `link_confidence`（数字）
- `link_reason`（长文本）
- `needs_confirmation`（复选框/布尔）
- `bundle_id`（文本）
- `message_ids`（长文本）
- `bundle_status`（单选：pending_media / single / merged）
- `bundle_window_seconds`（数字）

建议去重键：
- `session_id`
- `bundle_id`

## 3. 试衣素材索引

用于沉淀图片、语音等素材与主记录映射（可选；v2 主表 `media_urls` 已能承载基础需求）。

建议字段：
- `media_id`（文本）
- `session_id`（文本）
- `product_code`（文本）
- `media_type`（单选：tag_image / fitting_image / voice / other）
- `media_url`（超链接/文本）
- `media_source`（文本）
- `message_id`（文本）
- `captured_at`（日期时间）
- `ocr_confidence`（数字，v2 从主表迁来）
- `asr_confidence`（数字，v2 新增）

建议去重键：
- `media_id`

## 4. 线索回访表

用于沉淀基于试衣反馈生成的后续回访线索。

建议字段：
- `线索ID`（文本）
- `来源试衣记录ID`（文本）
- `日期`（日期）
- `导购名称`（文本）
- `导购open_id`（文本）
- `会员尾号`（文本）
- `试穿单品`（文本）
- `回访原因`（长文本）
- `建议触达时机类型`（单选：next_day / same_week / event_trigger）
- `计划触达时间`（日期时间）
- `建议回访内容`（长文本）
- `推荐动作`（文本）
- `优先级`（单选：高 / 中 / 低）
- `状态`（单选：pending / waiting_trigger / done / cancelled）

建议去重键：
- `线索ID`

必填校验字段（v2，8 个）：
- `线索ID`
- `来源试衣记录ID`
- `日期`
- `导购open_id`
- `会员尾号`
- `试穿单品`
- `回访原因`
- `状态`

## 5. 会员画像表

用于沉淀会员长期可复用的偏好与经营信息。

建议字段（v2 调整）：
- `会员尾号`（文本）
- `穿着偏好`（长文本，v2 合并 风格+版型）
- `尺码偏好`（文本）
- `颜色偏好`（长文本）
- `面料偏好`（长文本）
- `价格敏感度`（文本）
- `常见未购原因`（长文本）
- `体型备注`（长文本）
- `最近更新时间`（日期）
- `最近来源试衣记录`（文本）
- `画像置信度`（单选：高 / 中 / 低）

建议去重键：
- `会员尾号`

必填校验字段：
- `会员尾号`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

## 6. 提醒派发表（v2 新增）

由 `lobster-followup-reminder-dispatch` 维护，从 v1 回访表分离。

建议字段：
- `提醒ID`（文本）
- `来源线索ID`（文本）
- `导购open_id`（文本）
- `计划提醒时间`（日期时间）
- `是否已提醒`（复选框/布尔）
- `实际提醒时间`（日期时间）
- `提醒渠道`（单选：feishu_card / feishu_dm）
- `状态`（单选：scheduled / sent / cancelled / failed）

建议去重键：
- `提醒ID`

必填校验字段：
- `提醒ID`
- `来源线索ID`
- `导购open_id`
- `计划提醒时间`
- `是否已提醒`
- `状态`

## 7. _系统配置（v2 新增）

binding 自举入口，详见 [`binding-config-protocol.md`](../../references/binding-config-protocol.md)。

建议字段：
- `config_key`（文本）
- `config_value`（长文本，JSON 字符串）
- `schema_version`（文本）
- `updated_at`（日期时间）
- `updated_by_operator_id`（文本）

建议去重键：
- `config_key`

## 建表原则

- 先建最小可用字段，不在 bootstrap 阶段引入复杂公式字段
- 字段类型以当前飞书 Base 可稳定写入为优先
- 强枚举字段（`try_on_result` / `category` / `状态` / `画像置信度`）建议建为单选字段，预设选项
- 真实写入前，以上游 contract 和 sync 规则为准
- 若已有历史表结构，不强制重建，优先补齐缺失字段
- required 字段缺失时，应阻断正式写入，而不是继续部分落表
