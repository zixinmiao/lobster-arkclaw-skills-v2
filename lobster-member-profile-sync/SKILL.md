---
name: lobster-member-profile-sync
description: 基于试衣反馈补充会员画像，并同步写入独立的会员画像表。适用于已识别会员身份，需要稳定落会员画像必填字段的场景。v2 相对 v1：合并"风格偏好"+"版型偏好"→"穿着偏好"；删除"搭配需求"。
---

# 会员画像补充与同步（v2）

## 目标
从试衣反馈里抽取可长期复用的会员信息，沉淀为独立画像表。

## 字段 contract
写表前必须先读取 [`../references/bitable-field-contract.md`](../references/bitable-field-contract.md)，确认当前目标字段语义、最小可写字段集和可疑字段。

## binding 协议
按 [`../references/binding-config-protocol.md`](../references/binding-config-protocol.md) 复用项目级 binding。

## 自动触发规则
以下情况默认应触发本 skill：
- `member_mobile_last4` 非空（v2：替代 v1 的 `is_member = true` 判断）
- 已有明确会员身份，且本次试衣产生了可沉淀信息

## 最低同步要求

即使当前证据不足以写满画像，也应尽量补最小画像更新：
- `会员尾号`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

## 输入

- `fitting_record`
- `existing_member_profile`（可选）
- `current_time`（可选）

## 输出目标

返回 `result`：

- `profile_table_patch`
- `profile_sync_decision`
- `validator_result`（v2 新增）

### profile_table_patch

v2 画像表字段（11 字段）：
- `会员尾号`
- `穿着偏好`（v2 合并：风格偏好 + 版型偏好）
- `尺码偏好`
- `颜色偏好`
- `面料偏好`
- `价格敏感度`
- `常见未购原因`
- `体型备注`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

> v2 变化：
> - 合并：`风格偏好` + `版型偏好` → `穿着偏好`（两者抽出来语义高度重叠）
> - 删除：`搭配需求`（可由 `liked_points` 二次衍生）

### required 字段

正式写入前，以下字段必须全部非空：
- `会员尾号`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

校验委托给 `lobster-input-validator`。

## 写表约束

- `会员尾号` 是当前最稳的去重键；没有明确尾号时，不要硬写长期画像
- `最近来源试衣记录` 应直接引用主表 `session_id`
- `最近更新时间` 建议统一写 `YYYY-MM-DD` 或稳定日期时间格式
- `画像置信度` 必须是强枚举（`高` / `中` / `低`）+ 可附说明
- 若只有单次弱信号，优先把内容收敛到 `体型备注 / 常见未购原因 / 画像置信度`，不要一次写满所有偏好字段
- 识别到会员后，不允许静默跳过；至少要输出一份 `profile_table_patch` 或 `profile_sync_pending_reason`

## 字段填充建议（来自试衣反馈）

| 主表字段 | 画像表对应字段 |
|---|---|
| `body_effect_desc` + `fit_feedback` | `穿着偏好` / `体型备注` |
| `liked_points` | `穿着偏好` / `颜色偏好` / `面料偏好`（按 liked_points 内容分流） |
| `disliked_points` + `not_buy_reason` | `常见未购原因` |
| `tag_price` + `not_buy_reason`（含价格相关） | `价格敏感度` |
| `size` | `尺码偏好` |
| `color` | `颜色偏好` |
| `fabric` | `面料偏好` |
| `category` | （不直接对应；可作为偏好聚合维度） |

## 输出格式

```json
{
  "result": {
    "profile_table_patch": {
      "会员尾号": "0832",
      "穿着偏好": "倾向合体、有肩线、收腰；避免 oversize",
      "尺码偏好": "S（155/80A）",
      "颜色偏好": "藏青上身效果未满意，可继续探索",
      "最近更新时间": "2026-04-28",
      "最近来源试衣记录": "session_xxx",
      "画像置信度": "中（单次样本，建议持续补充）"
    },
    "profile_sync_decision": "...",
    "validator_result": {"can_write": true}
  }
}
```
