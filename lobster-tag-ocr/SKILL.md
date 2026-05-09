---
name: lobster-tag-ocr
description: 从吊牌图片中提取商品结构化信息。适用于导购上传吊牌图、价签图、衣物标签图时，识别商品编号、名称、颜色、尺码、面料、品类、吊牌价等字段，并为后续试衣记录沉淀做准备。该 skill 只负责输出商品候选信息，不直接推断成交、试穿结果或正式入库结论。v2 相对 v1 增加 fabric / category 字段。
---

# 吊牌信息识别（v2 字段扩展）

从图片中提取商品字段。

## 输出目标

输出 `result`，尽量包含：

| 字段 | 说明 |
|---|---|
| `product_name` | 商品名称 |
| `product_code` | 货号 / 款号 |
| `sku` | SKU（如吊牌印有） |
| `color` | 颜色 |
| `size` | 尺码 |
| `fabric` | 面料（v2 新增；常见值：棉、麻、羊毛、聚酯纤维、真丝、混纺等） |
| `category` | 品类（v2 新增；常见值：上装 / 下装 / 连衣裙 / 外套 / 配饰） |
| `tag_price` | 吊牌价 |
| `ocr_confidence` | 识别置信度 0~1 |
| `low_confidence_fields` | 数组；标记低置信度的字段名 |
| `raw_text` | 原始 OCR 文本（用于排查） |
| `missing_fields` | 数组；列出未识别字段 |

## 执行规则

- 优先识别明确印刷字段
- 多张吊牌图时，按图片分别识别，再汇总
- 对编号相近但不确定的值，不要猜测；写入 `low_confidence_fields`
- 若价格、颜色、尺码存在多个候选，在 `notes` 或 `raw_text` 中保留原始线索
- 单张吊牌图默认只输出商品候选信息，不得单独推断成交、已购买、已完成试穿记录等业务结论
- 若缺少后续文本或语音反馈，应将结果视为待补充输入，而不是正式记录
- `category` 推断优先级：吊牌明确印刷品类 > 商品名称关键词推断（如"裙"→连衣裙）> 文本上下文
- `fabric` 仅采纳吊牌明确印刷的成分信息；不要从颜色/视觉自行推断材质

## 低置信度处理

- `ocr_confidence < 0.6` 或单字段置信度低时，应输出 `low_confidence_fields`
- 上游若决定走兜底交互，可调用 `lobster-tag-ocr-retry` 引导导购重拍

## 输出格式

```json
{
  "result": {
    "product_name": "MO&Co.牛仔裤",
    "product_code": "EBF2JEN027-S97",
    "color": "牛仔蓝",
    "size": "26",
    "fabric": "棉 98% 氨纶 2%",
    "category": "下装",
    "tag_price": 1699,
    "ocr_confidence": 0.88,
    "low_confidence_fields": [],
    "missing_fields": []
  }
}
```
