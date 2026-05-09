# bitable field contract (v2)

本文件定义"导购小龙虾"相关飞书多维表格的目标字段语义，用于保证不同 open claw 在写表时对必填字段有一致理解，并在写入前做强制校验。

## 使用原则
- 本文件描述的是应有字段 / 目标 schema，不是线上实时状态。
- 线上表里实际已有字段，必须在运行时通过飞书 `field-list` 读取。
- 写表前必须先做字段检查，判断：`existing_fields` / `missing_required_fields` / `missing_recommended_fields` / `suspicious_fields`。
- 只要 `missing_required_fields` 非空，就不得正式写入。
- v2 中，统一字段校验由 `lobster-input-validator` 承担。

---

# 1. 试衣商品记录（主表）

## 1.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| session_id | 单条试衣记录唯一标识 | required | 统一生成的记录 ID | 手机号、open_id、商品名 |
| guide_name | 导购展示姓名 | required | 张三、Luna | open_id、手机号 |
| fitting_at | 试衣发生时间（datetime） | required | `2026/4/29 10:20` | open_id |
| store_name | 门店名称 | optional | 杭州万象城店 | open_id |
| operator_id | 导购飞书 open_id | required | `ou_xxx` | 顾客手机号、姓名 |
| member_mobile_last4 | 会员手机号后四位（v2 中：非空即视为会员） | optional | `0832` | 非数字文本 |
| product_code | 商品货号 / 款号 | required | `EBF2JEN027-S97` | 导购 ID |
| product_name | 商品名称 | required | `MO&Co.牛仔裤` | 历史上下文里的其他商品 |
| color | 商品颜色 | optional | 牛仔蓝色 | open_id |
| size | 商品尺码 | optional | `165/66A(26)` | open_id |
| fabric | 面料（v2 新增） | optional | 棉、羊毛、聚酯纤维 | open_id |
| category | 品类（v2 新增） | optional | 上装 / 下装 / 连衣裙 / 外套 / 配饰 | open_id |
| tag_price | 吊牌价 | optional | `1699` | 非价格文本 |
| try_on_result | 试穿结果（强枚举） | required | `合适` / `待考虑` / `未成交` / `已购` | 自由文本、纯图片猜测 |
| body_effect_desc | 上身体型呈现 | recommended | 显瘦、肩窄、显高 | open_id |
| fit_feedback | 版型反馈 | recommended | 腰部偏紧、袖长合适 | open_id |
| liked_points | 喜欢点 | recommended | 面料舒服、版型挺括 | open_id |
| disliked_points | 不喜欢点 | recommended | 颜色偏深 | open_id |
| not_buy_reason | 最终未购买原因 | optional | 有同款 / 预算卡点 | open_id |
| followup_intent | 是否有后续回访意向 | optional | 到货通知我 | open_id |
| raw_notes | 原始文字信息 | required | 原始试衣描述文本 | 空值 |
| media_urls | 关联素材 URL（v2：图片 URL 直挂主表） | optional | 多图 URL 列表 | open_id |

## 1.2 关键硬规则
- `guide_name` 只能写姓名/展示名，不能写 open_id
- `operator_id` 必须直接透传飞书 sender open_id，不依赖模型提取
- `session_id` 是主表唯一标识字段，写表时必须稳定生成
- `member_mobile_last4` 非空 → 视为会员；不再单独维护 `is_member` 字段
- `product_name` / `product_code` 必须优先取本次识别结果，不能被历史上下文商品覆盖
- `try_on_result` 必须取自枚举集，不允许自由文本
- `fitting_at` 是 datetime 单字段，不再拆 `fitting_date` + `fitting_time`

## 1.3 反馈四字段填充原则（v2 强调）
`body_effect_desc` / `fit_feedback` / `liked_points` / `disliked_points` 不依赖导购口语自行区分，而是由 AI 基于导购原始输入主动做语义拆分：
- 能拆出哪个就写哪个
- 不强求每次写满四个字段
- 不允许合并到单字段，因为多维度独立的字段对后续会员画像聚合分析有独立价值（喜欢点/不喜欢点是品类信号、上身效果是体型信号、版型反馈是产品信号）

## 1.4 最小可写字段集（v2，8 个 required）
- `session_id`
- `guide_name`
- `fitting_at`
- `operator_id`
- `product_code`
- `product_name`
- `try_on_result`
- `raw_notes`

## 1.5 v2 相对 v1 的字段变化

**删除（5）**：`source_bundle_id` / `source_message_ids` / `source_channel` / `session_status` / `ocr_confidence`（运维/状态字段，业务表不承载）

**合并（2 → 1）**：`fitting_date` + `fitting_time` → `fitting_at`

**简化（1）**：`is_member` 删除；`member_mobile_last4` 非空即会员

**新增（3）**：`fabric` / `category` / `media_urls`

**调整**：`try_on_result` 从自由文本改为强枚举

---

# 2. 线索回访表

## 2.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| 线索ID | 回访线索唯一标识 | required | `lead_xxx` | 手机号直写 |
| 来源试衣记录ID | 来源试衣记录 ID | required | `session_xxx` | 商品名 |
| 日期 | 线索生成日期 | required | `2026-04-28` | open_id |
| 导购名称 | 导购展示姓名 | recommended | 子心 | open_id |
| 导购open_id | 导购飞书 open_id | required | `ou_xxx` | 姓名 |
| 会员尾号 | 会员手机号后四位 | required | `2363` | 完整手机号 |
| 试穿单品 | 试穿单品（拼接 product_name + product_code） | required | `MO&Co.牛仔裤 EBF2JEN027-S97` | open_id |
| 回访原因 | 回访原因 | required | 高意向，考虑中，适时跟进 | open_id |
| 建议触达时机类型 | 建议触达时机类型 | recommended | `next_day` / `same_week` / `event_trigger` | 任意自由文本 |
| 计划触达时间 | 计划触达时间 | optional | `2026/4/1 18:00` | open_id |
| 建议回访内容 | 建议回访内容文案 | recommended | 回访建议文案 | open_id |
| 推荐动作 | 推荐动作 | recommended | 主动联系询问考虑进度 | open_id |
| 优先级 | 优先级 | recommended | `高` / `中` / `低` | open_id |
| 状态 | 状态 | required | `pending` / `waiting_trigger` / `done` / `cancelled` | 任意混乱状态值 |

> v1 中的 `是否已提醒` / `提醒时间` 在 v2 已迁出至"提醒派发表"，由 `lobster-followup-reminder-dispatch` 维护。

## 2.2 最小可写字段集（v2，8 个 required）
- `线索ID`
- `来源试衣记录ID`
- `日期`
- `导购open_id`
- `会员尾号`
- `试穿单品`
- `回访原因`
- `状态`

---

# 3. 会员画像表

## 3.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| 会员尾号 | 会员手机号的后四位数字 | required | `2291` | open_id |
| 穿着偏好 | 风格 + 版型偏好（v2 合并） | optional | 倾向利落、合体、有肩线、收腰；避免 oversize | open_id |
| 尺码偏好 | 尺码偏好 | optional | S码（155/80A） | open_id |
| 颜色偏好 | 颜色偏好 | optional | 偏好藏青、米白；藏青上身效果未满意，可继续探索 | open_id |
| 面料偏好 | 面料偏好 | optional | 羊毛、棉麻 | open_id |
| 价格敏感度 | 价格敏感度 | optional | 可接受 1500-3000 | open_id |
| 常见未购原因 | 常见未购原因 | optional | 上身效果不满意（风格不符） | open_id |
| 体型备注 | 体型备注 | optional | `155/80A（S）`、肩窄、腰细 | open_id |
| 最近更新时间 | 最近更新时间 | required | `2026-04-28` | open_id |
| 最近来源试衣记录 | 来源试衣记录中的 sessionID | required | `session_xxx` | 商品名 |
| 画像置信度 | 画像置信度 | required | `高` / `中` / `低` + 简要说明 | 任意无约束文本 |

> v2 变化：
> - 合并：`风格偏好` + `版型偏好` → `穿着偏好`
> - 删除：`搭配需求`（可由 `liked_points` 二次衍生）

## 3.2 最小可写字段集
- `会员尾号`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

---

# 4. 提醒派发表（v2 新增分离）

由 `lobster-followup-reminder-dispatch` 维护。从 v1 回访表分离出来，避免回访表 required 过多。

## 4.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| 提醒ID | 提醒唯一标识 | required | `remind_xxx` | open_id |
| 来源线索ID | 来源回访线索 ID | required | `lead_xxx` | 商品名 |
| 导购open_id | 接收提醒的导购 open_id | required | `ou_xxx` | 姓名 |
| 计划提醒时间 | 计划提醒时间 | required | `2026/4/1 17:00` | open_id |
| 是否已提醒 | 是否已提醒 | required | `true` / `false` | 文本"是/否" |
| 实际提醒时间 | 实际提醒时间 | optional | `2026/4/1 17:02` | open_id |
| 提醒渠道 | 提醒渠道 | optional | `feishu_card` / `feishu_dm` | open_id |
| 状态 | 状态 | required | `scheduled` / `sent` / `cancelled` / `failed` | 任意混乱状态值 |

## 4.2 最小可写字段集
- `提醒ID`
- `来源线索ID`
- `导购open_id`
- `计划提醒时间`
- `是否已提醒`
- `状态`

---

# 5. 试衣Session索引

用于追踪 session 生命周期和 bundle 归属。

| 字段名 | 含义 | 必需性 |
|---|---|---|
| session_id | session 唯一标识 | required |
| link_action | 链接动作（new / append / merge） | optional |
| link_confidence | 关联置信度 | optional |
| link_reason | 关联理由 | optional |
| needs_confirmation | 是否需要人工确认 | optional |
| bundle_id | bundle 标识 | optional |
| message_ids | 关联消息 IDs | optional |
| bundle_status | bundle 状态（pending_media / single / merged） | optional |
| bundle_window_seconds | 归并时间窗 | optional |

---

# 6. 试衣素材索引

用于沉淀图片、语音等素材与主记录映射（如果选择保留素材表；v2 默认主表 `media_urls` 已能承载基础需求，本表为可选）。

| 字段名 | 含义 | 必需性 |
|---|---|---|
| media_id | 素材 ID | required |
| session_id | 关联试衣记录 | required |
| product_code | 关联商品 | optional |
| media_type | `tag_image` / `fitting_image` / `voice` / `other` | required |
| media_url | 素材 URL | optional |
| media_source | 来源 | optional |
| message_id | 关联消息 ID | optional |
| captured_at | 采集时间 | optional |
| ocr_confidence | OCR 置信度（v2 从主表迁来） | optional |
| asr_confidence | ASR 置信度（v2 新增） | optional |

---

# 7. 系统配置表 `_系统配置`（v2 新增）

binding 自举入口，详见 [`binding-config-protocol.md`](binding-config-protocol.md)。

| 字段名 | 含义 | 必需性 |
|---|---|---|
| config_key | 配置键 | required |
| config_value | 配置值（JSON 字符串） | required |
| schema_version | schema 版本 | required |
| updated_at | 更新时间 | required |
| updated_by_operator_id | 更新人 open_id | recommended |

---

# 8. 运行时字段检查规则

写表前，skill（或统一委托给 `lobster-input-validator`）必须先读取飞书真实字段，并输出以下四类结果：
- `existing_fields`
- `missing_required_fields`
- `missing_recommended_fields`
- `suspicious_fields`

## can_write 判定
- 缺 `required` 字段 → `can_write = false`
- 仅缺 `recommended` 字段 → `can_write = true`，但要提示建议补字段
- 存在 `suspicious_fields` → 先人工确认或先治理后写

---

# 9. 当前收敛原则

v2 的优先目标是写入率：
- required 字段大幅收敛，避免阻断
- recommended 字段保留分析价值，但不阻断写入
- 字段拆分原则不变：能由 AI 主动做语义拆分的多维字段（如反馈四字段）保留独立性
- 字段类型口径统一（datetime / 强枚举 / 布尔），降低后续聚合分析成本
