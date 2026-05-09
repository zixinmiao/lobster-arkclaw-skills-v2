# ArkClaw 标准 Skill 目录（v2）

## 输入采集
- [`lobster-fitting-input-pipeline/SKILL.md`](lobster-fitting-input-pipeline/SKILL.md) — 消息归并、session 判断、合并字段、`pending_media` 判定一站式
- [`lobster-voice-transcribe/SKILL.md`](lobster-voice-transcribe/SKILL.md) — 音频附件 → transcript 文本（协议层，由 connector 实现 ASR）
- [`lobster-fitting-voice-extract/SKILL.md`](lobster-fitting-voice-extract/SKILL.md) — transcript → 结构化字段
- [`lobster-tag-ocr/SKILL.md`](lobster-tag-ocr/SKILL.md) — 吊牌图片 → 商品字段
- [`lobster-tag-ocr-retry/SKILL.md`](lobster-tag-ocr-retry/SKILL.md) — OCR 低置信度兜底引导
- [`lobster-member-identity-parse/SKILL.md`](lobster-member-identity-parse/SKILL.md) — 会员身份与尾号识别

## 状态管理
- [`lobster-fitting-draft-manager/SKILL.md`](lobster-fitting-draft-manager/SKILL.md) — 试衣草稿状态机
- [`lobster-fitting-card-fill/SKILL.md`](lobster-fitting-card-fill/SKILL.md) — 飞书卡片预填与确认
- [`lobster-input-validator/SKILL.md`](lobster-input-validator/SKILL.md) — 统一前置 required 字段校验

## 表写入
- [`lobster-fitting-bitable-bootstrap/SKILL.md`](lobster-fitting-bitable-bootstrap/SKILL.md) — 建表与 binding 初始化（含持久化协议）
- [`lobster-fitting-bitable-sync/SKILL.md`](lobster-fitting-bitable-sync/SKILL.md) — 主表写入
- [`lobster-followup-lead-sync/SKILL.md`](lobster-followup-lead-sync/SKILL.md) — 回访线索写入
- [`lobster-member-profile-sync/SKILL.md`](lobster-member-profile-sync/SKILL.md) — 会员画像写入

## 后置应用
- [`lobster-followup-reminder-dispatch/SKILL.md`](lobster-followup-reminder-dispatch/SKILL.md) — 回访提醒派发（维护提醒派发表）
- [`lobster-fitting-query/SKILL.md`](lobster-fitting-query/SKILL.md) — 试衣台账查询
- [`lobster-fitting-daily-summary/SKILL.md`](lobster-fitting-daily-summary/SKILL.md) — 每日汇总
