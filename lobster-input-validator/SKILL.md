---
name: lobster-input-validator
description: 写表前的统一 required 字段校验。适用于试衣主表、回访线索表、会员画像表、提醒派发表等多张表的写入前校验，把 v1 中散落在 record-merge / draft-manager / bitable-sync 三处的校验逻辑收口到一个 skill。v2 新增。
---

# 统一前置字段校验（v2 新增）

## 为什么独立成 skill

v1 中"required 字段校验"逻辑散落在三个 skill：
- `lobster-fitting-record-merge`：判断主表 required
- `lobster-fitting-draft-manager`：判断 write_gate
- `lobster-fitting-bitable-sync`：写入前再校验一次

加上回访表、会员画像表各自又有一套规则。结果是：
- 规则在不同 skill 间漂移
- 同一字段在不同 skill 中"必需性"不一致
- 维护成本高

v2 把校验逻辑收口到本 skill，所有写表 skill 在写入前调用一次。

## 输入

- `target_table`：目标表名，从枚举中选：
  - `试衣商品记录`
  - `线索回访表`
  - `会员画像表`
  - `提醒派发表`
- `payload`：拟写入的字段对象
- `field_sources`（可选但强烈建议）：字段来源对象，用于判断 required 字段是否可信，例如 `{"guide_name":"sender_profile_confirmed","operator_id":"message_metadata"}`
- `binding_field_map`（可选）：从 binding 中读到的真实字段映射，用于检测线上字段缺失
- `strict_mode`（可选）：默认 `true`；`false` 时仅警告不阻断

## 输出目标

返回 `result`：

| 字段 | 说明 |
|---|---|
| `can_write` | 布尔；缺 required 字段时为 false |
| `existing_fields` | 数组；payload 中已有的字段名 |
| `missing_required_fields` | 数组；缺失的 required 字段 |
| `missing_recommended_fields` | 数组；缺失的 recommended 字段 |
| `suspicious_fields` | 数组；存在但值可疑的字段（如 guide_name 写成 open_id） |
| `enum_violations` | 数组；强枚举字段值不在合法集合中（如 category 写了非枚举值） |
| `validation_summary` | 字符串；可读摘要，用于上游打日志或发卡片 |

## 各表 required 集合（来自 contract）

### 试衣商品记录（v2.1，7 个 required）
- `session_id`
- `guide_name`
- `fitting_at`
- `operator_id`
- `product_code`
- `product_name`
- `raw_notes`

### 线索回访表（v2，8 个 required）
- `线索ID`
- `来源试衣记录ID`
- `日期`
- `导购open_id`
- `会员尾号`
- `试穿单品`
- `回访原因`
- `状态`

### 会员画像表
- `会员尾号`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

### 提醒派发表
- `提醒ID`
- `来源线索ID`
- `导购open_id`
- `计划提醒时间`
- `是否已提醒`
- `状态`

## 校验维度

### 1. 必需性
- 缺 required → `can_write = false`
- 缺 recommended → `can_write = true`，但记录在 `missing_recommended_fields`
- required 字段有值但来源不可信 → 视为未通过校验，写入 `suspicious_fields`，`can_write = false`

### 1.1 required 字段可信来源

`guide_name` 必须来自以下来源之一：
- `current_message_explicit`
- `sender_profile_confirmed`
- `guide_profile_confirmed`
- `card_confirmed`
- `trusted_contact`

`operator_id` 必须来自 `message_metadata` 或 connector 透传的 sender open_id。

若没有提供 `field_sources`，validator 仍必须基于值本身做可疑值检测；若值看起来像占位词或角色名，必须阻断。

### 2. 强枚举值
| 字段 | 合法集合 |
|---|---|
| `状态`（回访表） | `pending` / `waiting_trigger` / `done` / `cancelled` |
| `状态`（提醒表） | `scheduled` / `sent` / `cancelled` / `failed` |
| `category` | `上装` / `下装` / `连衣裙` / `外套` / `配饰` |
| `画像置信度` | `高` / `中` / `低`（可附说明） |

> 枚举集合的权威来源是 [`../references/schema.json`](../references/schema.json)；本表是注解版本，发生冲突以 schema.json 为准。

枚举字段值不在集合内 → 写入 `enum_violations`，且 `can_write = false`

### 3. 可疑值检测（suspicious_fields）

| 字段 | 可疑模式 |
|---|---|
| `guide_name` | 值以 `ou_` 开头（疑似写成 open_id）；值为 `导购` / `店员` / `销售` / `营业员` / `未知` / `默认导购` / `测试导购` / `N/A` / 空字符串；值像手机号 |
| `operator_id` | 值不以 `ou_` 开头且不是已知员工 ID 格式（疑似写成姓名） |
| `member_mobile_last4` | 长度不为 4 / 含非数字 |
| `product_code` | 长度过短（< 4） |
| `fitting_at` | 时间格式不合飞书 datetime 规范 |

### 4. 线上字段一致性（如提供 binding_field_map）
- 检测 payload 中是否存在线上不存在的字段（防止字段名漂移）
- 检测线上 required 字段是否被 payload 覆盖

### 5. 缺失字段追问输出

当 `guide_name` 缺失或可疑时，上游必须生成同一类追问，不允许静默写占位值：

```json
{
  "field": "guide_name",
  "action": "ask_user",
  "message": "请补充导购姓名，之后同一导购可自动沿用。"
}
```

## 调用位置

| 上游 skill | 调用时机 |
|---|---|
| `lobster-fitting-input-pipeline` | 输出 fitting_records 前 |
| `lobster-fitting-card-fill` | 卡片提交时 |
| `lobster-fitting-bitable-sync` | 写主表前最后一道 |
| `lobster-followup-lead-sync` | 写回访表前 |
| `lobster-member-profile-sync` | 写画像表前 |
| `lobster-followup-reminder-dispatch` | 写提醒表前 |

## 输出格式

```json
{
  "result": {
    "can_write": false,
    "existing_fields": ["session_id", "guide_name", "..."],
    "missing_required_fields": ["fitting_at"],
    "missing_recommended_fields": [],
    "suspicious_fields": [
      {"field": "guide_name", "value": "ou_xxx", "reason": "looks like open_id"}
    ],
    "enum_violations": [
      {"field": "category", "value": "T恤", "allowed": ["上装", "下装", "连衣裙", "外套", "配饰"]}
    ],
    "validation_summary": "缺少 1 个 required 字段；1 个枚举违规；1 个可疑值"
  }
}
```
