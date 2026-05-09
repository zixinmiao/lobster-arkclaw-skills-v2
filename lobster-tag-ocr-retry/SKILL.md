---
name: lobster-tag-ocr-retry
description: 当吊牌 OCR 置信度过低或关键字段缺失时，生成引导导购重拍的飞书消息文案。适用于 OCR 失败兜底场景，避免直接以低质量数据写入主表。v2 新增。
---

# 吊牌 OCR 兜底引导（v2 新增）

## 为什么需要这个 skill

v1 的 `lobster-tag-ocr` 没有失败处理；置信度低时只输出 `ocr_confidence` 数字，不主动引导。结果是：
- 低质量 OCR 直接进入下游，污染主表
- 或者上游静默丢弃，导购不知道为什么没生成记录

v2 通过本 skill 让"OCR 不行"变成可观测、可解决的对话。

## 触发条件

由上游 workflow 在以下情况调用：
- `ocr_result.ocr_confidence < 0.6`
- `ocr_result.low_confidence_fields` 包含关键字段（`product_code` / `product_name`）
- `ocr_result.missing_fields` 包含 `product_code`
- `ocr_result.raw_text` 为空或不可识别

## 输入

- `ocr_result`：上游 `lobster-tag-ocr` 的输出
- `bundle_id` / `draft_id`：用于关联当前会话上下文
- `retry_count`（可选）：当前是第几次重试，影响话术
- `guide_name`（可选）：用于人称化文案

## 输出目标

返回 `result`：

| 字段 | 说明 |
|---|---|
| `retry_action` | `ask_reshoot` / `ask_manual_input` / `give_up` |
| `card_payload` | 飞书卡片 payload（结构化），用于推送给导购 |
| `prompt_text` | 纯文本兜底（如不支持卡片） |
| `low_confidence_summary` | 低置信度字段及原因列表 |
| `next_state` | 草稿应进入的状态（建议交给 draft-manager） |

## 文案规则

- **首次兜底**：`retry_action = ask_reshoot`，提示"吊牌图不够清晰，麻烦把吊牌平铺、对焦后再拍一张"，附低置信度字段
- **重拍仍失败（retry_count ≥ 2）**：`retry_action = ask_manual_input`，提示"OCR 仍未识别清楚 `product_code`，麻烦直接发文字告诉我货号"
- **三次以上失败**：`retry_action = give_up`，建议人工介入或跳过 OCR、走纯文本路径
- 文案应说明**具体哪几个字段没识别清楚**，不要只说"OCR 失败"
- 不应一次列超过 3 个低置信度字段，避免信息过载

## 卡片字段建议（feishu interactive card）

```json
{
  "header": "吊牌识别置信度偏低",
  "fields": [
    {"label": "已识别", "value": "product_name: MO&Co.牛仔裤"},
    {"label": "不确定", "value": "product_code（疑似 EBF2JEN027 但末段模糊）"}
  ],
  "actions": [
    {"label": "重拍吊牌", "value": "reshoot"},
    {"label": "我直接打字告诉你货号", "value": "manual_input"}
  ]
}
```

## 执行规则

- 不在本 skill 中重新调用 OCR，只生成引导动作
- 不写入主表（写入决策由 draft-manager + bitable-sync 联合判断）
- 应在 `next_state` 字段建议草稿状态：`pending_media` 或 `pending_feedback`，让 draft-manager 衔接
- 重试上限：建议 3 次后切人工介入

## 输出格式

```json
{
  "result": {
    "retry_action": "ask_reshoot",
    "card_payload": {},
    "prompt_text": "吊牌看不清，能不能重拍一张？特别是货号那一行。",
    "low_confidence_summary": [
      {"field": "product_code", "reason": "末段模糊"}
    ],
    "next_state": "pending_media"
  }
}
```
