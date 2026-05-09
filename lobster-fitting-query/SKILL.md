---
name: lobster-fitting-query
description: 查询并汇总试衣主记录。适用于用户按日期、门店、导购、会员尾号、款号、颜色、尺码、面料、品类、试穿结果等条件查询试衣记录，或需要生成明细加统计摘要时使用。v2 字段集与主表 contract 对齐。
---

# 试衣单查询与统计（v2）

根据查询条件生成查询计划、结果结构和摘要。

## binding 协议
按 [`../references/binding-config-protocol.md`](../references/binding-config-protocol.md) 复用项目级 binding。

## 支持过滤条件（v2）

- `date`
- `date_range`
- `store_name`
- `guide_name`
- `operator_id`
- `member_mobile_last4`（v2：非空过滤等价于"会员"）
- `is_member_filter`：`true` / `false`（语义层；底层等价于 `member_mobile_last4` 非空判断）
- `product_code`
- `product_name`
- `color`
- `size`
- `fabric`（v2 新增）
- `category`（v2 新增；强枚举：上装 / 下装 / 连衣裙 / 外套 / 配饰）
- `has_followup_intent`（基于 followup_intent 字段非空判断；v2.1 替代原 try_on_result 过滤）
- `not_buy_reason_keyword`（v2.1 新增；按未购原因关键词模糊匹配）

## 查询模式

- `detail`：仅明细
- `summary`：仅汇总
- `detail_and_summary`：默认

## 输出目标

输出：
- `records`
- `summary`
- `summary_text`
- `query_status`
- `query_plan`

## 执行规则

- 若有真实表连接，则按 binding 中的 `field_map` 执行
- 若 binding `last_verified_at` 超时，先触发轻量校验
- 若当前只拿到数据样本或自然语言条件，则先生成标准查询条件与期望结果结构
- 结果为空时，明确返回空结果，不强行总结
- v2 删除字段（如 `is_member` / `source_bundle_id`）不应作为查询条件

## 输出格式

```json
{
  "records": [],
  "summary": {},
  "summary_text": "",
  "query_status": "success | empty | missing_binding",
  "query_plan": {}
}
```
