---
name: lobster-fitting-daily-summary
description: 汇总某日试衣录入情况并生成可直接发送的日报。适用于门店晚间复盘、店长查看当日录入单量、商品件数、会员单量、异常记录和导购表现时使用。v2 字段集与主表 contract 对齐。
---

# 试衣日报汇总（v2）

生成可直接发送的试衣日报。

## binding 协议
按 [`../references/binding-config-protocol.md`](../references/binding-config-protocol.md) 复用项目级 binding。

## 默认统计项

- `fitting_session_count`：当日试衣 session 数
- `fitting_item_count`：当日商品件数
- `member_session_count`：会员试衣 session 数（基于 `member_mobile_last4` 非空判断）
- `abnormal_record_count`：异常记录数
- `top_guides`：导购排行
- `top_products`：商品排行
- `category_distribution`（v2 新增）：品类分布
- `try_on_result_distribution`（v2 新增）：试穿结果分布（合适 / 待考虑 / 未成交 / 已购）

## 异常规则

- `missing_product_code`
- `missing_size`
- `missing_fabric`（v2 新增）
- `missing_category`（v2 新增）
- `low_ocr_confidence`：从 `试衣素材索引` 表中读取（v2 中 `ocr_confidence` 已从主表迁到素材表）
- `try_on_result` 为空（数据完整性异常）

## 输出目标

输出：
- `summary_result`
- `summary_text`
- `summary_status`

### summary_text

用业务可转发口径输出，优先包括：
- 日期
- 录入单量
- 商品件数
- 会员试衣单数
- 试穿结果分布（已购 X 单 / 待考虑 X 单 / 未成交 X 单）
- 品类分布
- 异常记录数
- 表现亮点
- 待补录提醒

## 执行规则

- 未指定日期时默认当天
- 有数据源就汇总真实数据；没有数据源就输出汇总模板和缺失项
- 不夸大亮点，不隐去异常
- 异常记录应给出具体 `session_id`，便于导购回查

## 输出格式

```json
{
  "summary_result": {},
  "summary_text": "",
  "summary_status": "success | empty | missing_binding"
}
```
