---
name: lobster-member-identity-parse
description: 识别试衣场景中的会员身份信息。适用于导购文字、语音转写、上下文中出现会员、手机号尾号、会员备注等线索时，判断是否为会员试衣，并提取会员尾号后四位与状态字段。v2 中：member_mobile_last4 非空即视为会员，不再单独维护 is_member 字段。
---

# 会员身份识别

识别是否为会员，并输出结构化结果。

## 输出目标

输出 `result`：

| 字段 | 说明 |
|---|---|
| `member_mobile_last4` | 字符串，4 位数字 |
| `member_identity_status` | `confirmed` / `mentioned_without_last4` / `not_member` / `unknown` |
| `member_notes` | 字符串 |
| `confidence` | 0 到 1 |

> v2 变化：删除 `is_member` 输出字段。判定规则改为"`member_mobile_last4` 非空 → 视为会员"，由下游统一判断。

## 判定规则

- 明确提到"会员"且给出尾号后四位 → `confirmed`
- 提到"会员"但没有尾号 → `mentioned_without_last4`
- 明确表示不是会员 → `not_member`
- 信息不足 → `unknown`
- 只提到数字但无足够上下文，不要直接认定是会员尾号
- transcript 中标记为 `low_confidence_numeric` 的尾号候选，应在 `member_notes` 备注，并把 `confidence` 降权

## 数字与会员尾号的区分

数字串可能是：
- 会员手机号尾号（4 位）
- 商品 SKU 末段（也常是 3-4 位数字）
- 价格、尺码

需要结合上下文：
- 出现"会员"、"会员卡"、"卡号"、"积分"、"尾号"等关键词附近的 4 位数字 → 倾向会员尾号
- 出现"码"、"号"、"￥"、"元"附近的数字 → 排除会员

## 输出格式

```json
{
  "result": {
    "member_mobile_last4": "0832",
    "member_identity_status": "confirmed",
    "member_notes": "...",
    "confidence": 0.92
  }
}
```
