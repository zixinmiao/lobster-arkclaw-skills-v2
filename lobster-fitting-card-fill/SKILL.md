---
name: lobster-fitting-card-fill
description: 根据 draft、OCR、voice-extract 和 input-pipeline 结果生成飞书试衣录入卡片的预填内容，并接收卡片确认后的结构化结果。适用于需要围绕 v2 主表字段做卡片确认提交的场景。
---

# 试衣录入卡片预填与确认

## 目标

把 draft 中的候选字段整理成飞书卡片可展示、可编辑、可提交的结构。

## 输入

- `draft_result`
- `pipeline_result`（v2 中替代 v1 的 merge_result）
- `ocr_result`
- `voice_extract_result`（v2 中替代 v1 的 stt_result）

## 卡片字段建议（v2 字段集）

### 商品信息区
- `product_name`
- `product_code`
- `color`
- `size`
- `fabric`（v2 新增）
- `category`（v2 新增）
- `tag_price`

低置信度字段（OCR 标记或 ASR 标记）应在卡片上以特别样式提醒，让导购优先纠正。

### 试穿反馈区
- `try_on_result`（强枚举下拉：合适 / 待考虑 / 未成交 / 已购）
- `body_effect_desc`
- `fit_feedback`
- `liked_points`
- `disliked_points`
- `not_buy_reason`
- `followup_intent`

四个反馈字段独立编辑，由 AI 预填后导购可逐项调整。

### 基础区
- `session_id`（不可编辑）
- `guide_name`
- `store_name`
- `fitting_at`（datetime；v2 合并字段）
- `member_mobile_last4`

## 提交规则

- 点击 `submit_record` 后，输出结构化确认结果
- 提交前必须经 `lobster-input-validator` 校验 required 字段；若缺失，返回阻断原因，不得直接写入
- 卡片提交结果应对齐 v2 主表保留字段（21 字段集），不再要求已删除的字段（`source_bundle_id` / `source_message_ids` / `is_member` 等）

## 低置信度处理

- 若上游标记 `low_confidence_fields` 包含数字字段（`product_code` / `tag_price` / `member_mobile_last4`），卡片应在这些字段旁加视觉提醒（如 ⚠️ 或红色边框）
- 提示文案如："这几个数字是从语音/图片识别的，麻烦核对下"
- 不强制导购纠正，但纠正后置信度提升，下游写入更稳

## 输出目标

返回 `result`：

- `card_payload`：飞书 interactive card 结构
- `card_state`：`pending_input` / `awaiting_confirm` / `submitted` / `cancelled`
- `submit_payload_schema`：提交后回包字段定义
- `blocking_rules`：阻断规则说明

## 输出格式

```json
{"result": {...}}
```
