---
name: lobster-followup-reminder-dispatch
description: 扫描线索回访表中的到期待触达记录，生成导购提醒内容，并维护独立的提醒派发表。适用于由统一定时任务周期性触发。v2 相对 v1：v1 把"是否已提醒/提醒时间"挂在回访表，v2 改为独立的提醒派发表，本 skill 维护这张表。
---

# 回访提醒派发（v2）

## 目标
把"每条线索手动配置定时提醒"改成"统一定时任务 + 扫描派发"，并把提醒状态持久化到独立的"提醒派发表"，避免污染回访线索表的主体字段。

## v2 关键改动

v1 中：
- "是否已提醒""提醒时间" 直接写在回访线索表
- 回访表 required 字段 14 个，写入率受影响

v2 中：
- 这两个字段从回访表迁出
- 独立创建"提醒派发表"，由本 skill 维护
- 一条回访线索可对应 0 或多条提醒记录（首次提醒、重复提醒、跨天提醒）

## 输入

- `current_time`
- `followup_leads`：从回访线索表读取的待派发线索数组
- `existing_reminders`（可选）：从提醒派发表读取的历史提醒
- `dispatch_window_minutes`（可选，默认 15）
- `dispatch_mode`（可选，默认 `scheduled_due`）
- `channel_context`（可选）

### followup_leads

v2 回访表字段集（14 字段；不含 v1 的"是否已提醒/提醒时间"）：
- `线索ID`
- `来源试衣记录ID`
- `日期`
- `导购名称`
- `导购open_id`
- `会员尾号`
- `试穿单品`
- `回访原因`
- `建议触达时机类型`
- `计划触达时间`
- `建议回访内容`
- `推荐动作`
- `优先级`
- `状态`

## 扫描规则

默认模式 `dispatch_mode = scheduled_due` 下，满足以下条件时进入派发候选：
- 回访表 `状态 = pending` 或 `waiting_trigger`
- 回访表 `计划触达时间 <= current_time`
- 提醒派发表中没有对应该 `线索ID` 的 `已提醒` 记录（避免重复发）

## 输出目标

返回 `result`：

- `due_reminders`：本次拟发送的提醒
- `reminder_table_patches`：写入提醒派发表的记录
- `lead_status_patches`：回写回访表的状态更新（仅状态变更，不再写"是否已提醒"）
- `dispatch_summary`

### due_reminders（用于实际发送）

每条尽量包含：
- `线索ID`
- `导购名称`
- `导购open_id`
- `target_chat_id`
- `reminder_title`
- `reminder_text`
- `priority`
- `send_now`
- `reason`

### reminder_table_patches（写入提醒派发表）

提醒派发表字段（v2 contract）：
- `提醒ID`：必填，建议格式 `remind_${linkid}_${timestamp}`
- `来源线索ID`：必填，引用回访表 `线索ID`
- `导购open_id`：必填
- `计划提醒时间`：必填
- `是否已提醒`：必填（创建时默认 false）
- `实际提醒时间`：可选
- `提醒渠道`：可选（`feishu_card` / `feishu_dm`）
- `状态`：必填，强枚举（`scheduled` / `sent` / `cancelled` / `failed`）

### lead_status_patches（回写回访表）

仅在以下情况回写回访表 `状态`：
- 提醒发送成功且业务认为线索已闭环 → `状态 = done`
- 提醒发送失败超过重试上限 → `状态 = cancelled`
- 否则不回写

> v1 中要回写 `是否已提醒` 与 `提醒时间`；v2 不再写这两个字段。

## 派发与回写规则

1. 扫描候选 → 生成 `due_reminders`
2. 若发送成功：
   - `reminder_table_patches` 中追加一条记录，`是否已提醒 = true`，`实际提醒时间 = now`，`状态 = sent`
   - 同步评估是否需要回写回访表 `状态`
3. 若发送失败：
   - `reminder_table_patches` 中追加一条记录，`状态 = failed`
   - 重试由调度层管理

## 校验

写入提醒派发表前调用 `lobster-input-validator(target_table=提醒派发表, payload)`，校验 required 字段。

## 输出格式

```json
{
  "result": {
    "due_reminders": [],
    "reminder_table_patches": [
      {
        "提醒ID": "remind_lead_xxx_20260509100000",
        "来源线索ID": "lead_xxx",
        "导购open_id": "ou_xxx",
        "计划提醒时间": "2026/4/29 18:00",
        "是否已提醒": true,
        "实际提醒时间": "2026/4/29 18:02",
        "提醒渠道": "feishu_card",
        "状态": "sent"
      }
    ],
    "lead_status_patches": [],
    "dispatch_summary": "..."
  }
}
```
