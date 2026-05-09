---
name: lobster-fitting-voice-extract
description: 把试衣场景的语音 transcript 文本结构化成业务字段。适用于已通过 lobster-voice-transcribe 完成 ASR 转写后的二次结构化抽取，提取导购、门店、客户、商品、颜色、尺码、面料、品类、价格、试穿反馈、跟进动作等字段。本 skill 是 v1 lobster-fitting-stt 的改名与职责对齐版本。
---

# 试衣语音文本结构化（v2 改名 + 职责对齐）

> v2 改动：原 `lobster-fitting-stt` 的命名误导（实际只做"transcript → 字段"，不做 ASR 转写）。改名为 `voice-extract`，职责与实际能力对齐；真正的音频→文本由 `lobster-voice-transcribe` 协议定义、由 connector 实现。

## 前置要求

本 skill 接收的输入应为已经过 ASR 转写的 transcript 文本。如果上游传入的是音频附件，应先经 `lobster-voice-transcribe`。

不要在仅因"当前是录音消息"就直接回复"暂时无法直接转写语音"——v2 已通过 `lobster-voice-transcribe` 协议补齐 ASR 链路。

## 输入

- `transcript`：必填，ASR 转写后的纯文本
- `transcript_segments`（可选）：分段转写，含 `low_confidence_numeric` 标记
- `asr_confidence`（可选）
- `context`（可选）：`store_name` / `guide_name` / 最近 open session 等
- `boost_terms`（可选）：业务热词

## 输出目标

输出一个 JSON 对象 `result`，优先包含：

| 字段 | 说明 |
|---|---|
| `transcript_cleaned` | 整理后的文本（去口语、重复、语气词；不改原意） |
| `guide_name` | 导购姓名 |
| `store_name` | 门店 |
| `customer_name` | 顾客称呼（如有） |
| `member_mobile_last4` | 会员尾号（不要自行定性，交由 member-identity-parse） |
| `fitting_at` | 试衣时间（datetime） |
| `products` | 数组；每项含 `product_name` / `product_code` / `color` / `size` / `fabric` / `category` / `tag_price` |
| `body_effect_desc` | 上身效果 |
| `fit_feedback` | 版型反馈 |
| `liked_points` | 喜欢点 |
| `disliked_points` | 不喜欢点 |
| `not_buy_reason` | 未购原因 |
| `followup_intent` | 后续意向 |
| `next_action` | 建议下一步动作 |
| `notes` | 其他备注 |
| `missing_fields` | 数组；列出未识别字段 |
| `low_confidence_fields` | 数组；标记数字类字段（`product_code` / `tag_price` / `member_mobile_last4`）由 ASR 推得且置信度低，提示下游卡片确认环节由导购纠正 |

## 执行规则

- 保留业务事实，不补造未提及信息
- 口语、重复、语气词可以整理，但不要改变原意
- 有多个商品时，按商品拆分到 `products`
- 识别不确定信息时，用 `unknown` 或放入 `notes`，并写入 `missing_fields`
- `guide_name` 只有在 transcript 明确出现姓名/展示名时才抽取；"我是导购"、"店员说"、"销售反馈"这类角色描述不得抽成 `guide_name`
- 不确定导购姓名时，不要输出 `guide_name: "导购"` 或 `unknown`；应留空并把 `guide_name` 写入 `missing_fields`
- 如果 transcript 中包含会员线索，**不要自行定性是否会员**，把候选信息保留到 `member_mobile_last4` / `notes`，交由 `lobster-member-identity-parse` 判定
- `transcript_segments` 中带 `low_confidence_numeric` 的段落，对应字段必须写入 `low_confidence_fields`
- 反馈类字段按四维拆分（与 input-pipeline 一致）：
  - `body_effect_desc`：体型呈现（"显瘦"、"显高"、"肩窄"）
  - `fit_feedback`：版型问题（"腰部偏紧"、"袖长合适"）
  - `liked_points`：主观喜欢（"喜欢颜色"、"面料舒服"）
  - `disliked_points`：主观不喜欢（"颜色偏深"、"版型太宽松"）

## 输出格式

```json
{
  "result": {
    "transcript_cleaned": "...",
    "guide_name": "...",
    "products": [
      {"product_name": "...", "product_code": "...", "color": "...", "size": "...", "fabric": "...", "category": "..."}
    ],
    "body_effect_desc": "...",
    "fit_feedback": "...",
    "liked_points": "...",
    "disliked_points": "...",
    "low_confidence_fields": ["product_code", "tag_price"],
    "missing_fields": []
  }
}
```
