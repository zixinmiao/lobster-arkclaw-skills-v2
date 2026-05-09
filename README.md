# lobster-arkclaw-skills-v2

导购试衣与飞书多维表格场景的 ArkClaw/OpenClaw 标准 Skill 集合。本仓库是 [v1](https://github.com/zixinmiao/lobster-arkclaw-skills) 的精简与重构版本。

## v2 相对 v1 的核心变化

- **字段精简**：主表 26 → 20 字段；主表 required 11 → 7；回访表 required 14 → 8
- **语音消息可处理**：新增 `lobster-voice-transcribe` 协议，把 ASR 能力下沉到 connector 层；原 `lobster-fitting-stt` 改名为 `lobster-fitting-voice-extract`，职责退化为 transcript 文本结构化
- **Binding 持久化协议化**：把"项目级共享 binding"从口头约定升级为明确协议；新增 `references/binding-config-protocol.md`，定义两层兜底（平台 KV + Base 内 `_系统配置` 表）
- **流水线收敛**：`bundle-router` + `record-merge` 合并为 `lobster-fitting-input-pipeline`，但保留 `pending_media` 状态机不丢
- **去掉低价值 skill**：删除 `photo-archive`、`script-generate`
- **新增**：`lobster-input-validator`（统一前置校验）、`lobster-tag-ocr-retry`（低置信度兜底）

## v2.1 增量（治建表不一致问题）

- **新增 [`references/schema.json`](references/schema.json)**：机器可读权威 schema；bootstrap 改 schema-driven，建完字段必须回读校验，根治"两个 agent 解读出不同字段集"
- **Canonical 字段命名强约束**：禁止 `guidename` / `guideName` / `导购姓名` 等同义变体替代 `guide_name`
- **删除 `try_on_result` 字段**：试穿结果由反馈四字段 + `not_buy_reason` + `followup_intent` 自然表达；主表不再单独维护强枚举结果字段

详见 [`MIGRATION_FROM_V1.md`](MIGRATION_FROM_V1.md)。

## Skills

详见 [`SKILLS_INDEX.md`](SKILLS_INDEX.md)。

## Workflow

详见 [`WORKFLOW.md`](WORKFLOW.md)：定义推荐的 skill 编排顺序、初始化绑定规则、必填字段校验和稳定写入链路。

## 关键文档

- [`references/bitable-field-contract.md`](references/bitable-field-contract.md)：三张业务表的字段 contract，定义必填字段、可选字段、允许/禁止写入内容、枚举值
- [`references/binding-config-protocol.md`](references/binding-config-protocol.md)：项目级 binding 持久化协议，规定 binding 数据结构、读写时序、两层兜底
- [`lobster-fitting-bitable-bootstrap/references/default-schema.md`](lobster-fitting-bitable-bootstrap/references/default-schema.md)：默认建表结构

## 当前收敛重点

- 写入率优先：required 字段大幅收敛，避免因为单字段缺失阻断整次写入
- 数据稳定性优先：单图无文本、`pending_media` 状态绝不强写主表
- 多导购共用同一套 Base：binding 必须可持久化、可复用、可校验
- skill 职责边界清晰：建表 / 写表 / 校验 / 状态管理分离
