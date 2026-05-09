---
name: lobster-followup-lead-sync
description: 根据试衣反馈判断是否生成回访线索，并同步写入线索回访表。适用于需要稳定落回访必填字段的场景。v2 相对 v1：required 字段从 14 收敛到 8（"是否已提醒/提醒时间"已迁出至提醒派发表）。
---

# 试衣回访线索生成与落表（v2）

## 目标
基于单条试衣记录，判断是否需要生成回访线索，并输出可直接写入"线索回访表"的结构化结果。

## 字段 contract
写表前必须先读取 [`../references/bitable-field-contract.md`](../references/bitable-field-contract.md)，确认当前目标字段语义、最小可写字段集和可疑字段。

## binding 协议
按 [`../references/binding-config-protocol.md`](../references/binding-config-protocol.md) 复用项目级 binding：
- 优先从平台 KV 读
- 兜底从 `_系统配置` 表读
- `last_verified_at` 超时则触发轻量校验

## 输入
- `fitting_record`
- `member_context`（可选）
- `store_context`（可选）
- `current_time`（可选）
- `campaign_context`（可选）

## 输出目标
返回 `result`：
- `followup_lead`
- `decision_reason`
- `validator_result`（v2 新增；调用 lobster-input-validator 后的结果）

### followup_lead
回访表写入字段（v2，14 字段，移除"是否已提醒/提醒时间"）：
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

### required 字段（v2，8 个）
正式写入前，以下字段必须全部非空（其余为 recommended，不阻断写入）：
- `线索ID`
- `来源试衣记录ID`
- `日期`
- `导购open_id`
- `会员尾号`
- `试穿单品`
- `回访原因`
- `状态`

> v2 改 recommended 的字段：`建议触达时机类型` / `计划触达时间` / `建议回访内容` / `推荐动作` / `优先级` / `导购名称`。这些字段缺失不阻断写入，但应在 `decision_reason` 中记录。

校验委托给 `lobster-input-validator`，本 skill 不重复校验逻辑。

## 生成规则

- 只有"有兴趣 + 未成交 + 原因明确 + 后续可被导购动作解决"时，优先生成线索
- 非会员（`member_mobile_last4` 为空）→ 跳过线索生成；会员画像可能仍需补，但回访线索不强生成
- `来源试衣记录ID` 应直接引用主表 `session_id`
- `导购名称` 来自 `guide_name`
- `导购open_id` 应来自 `operator_id`，不得留空
- `试穿单品` 应优先拼接 `product_name + product_code`
- `状态` 创建时默认优先写 `pending`；若是触发式线索可写 `waiting_trigger`
- `状态` 必须是强枚举（`pending` / `waiting_trigger` / `done` / `cancelled`）

## 与提醒派发的关系（v2 改动）

v1 中"是否已提醒""提醒时间"两个字段在回访表中维护，导致 required 字段过多。v2 把这两个字段迁出到独立的"提醒派发表"，由 `lobster-followup-reminder-dispatch` 维护：
- 本 skill 不再写"是否已提醒""提醒时间"
- 提醒派发表中的 `来源线索ID` 字段引用本回访表的 `线索ID`
- 一条回访线索可对应 0 或多条提醒派发记录

## 输出格式

```json
{
  "result": {
    "followup_lead": {
      "线索ID": "lead_xxx",
      "来源试衣记录ID": "session_xxx",
      "日期": "2026-04-28",
      "导购名称": "子心",
      "导购open_id": "ou_xxx",
      "会员尾号": "0832",
      "试穿单品": "MO&Co.牛仔裤 EBF2JEN027-S97",
      "回访原因": "高意向，考虑预算",
      "建议触达时机类型": "next_day",
      "计划触达时间": "2026/4/29 18:00",
      "建议回访内容": "...",
      "推荐动作": "主动联系询问考虑进度",
      "优先级": "中",
      "状态": "pending"
    },
    "decision_reason": "...",
    "validator_result": {"can_write": true}
  }
}
```
