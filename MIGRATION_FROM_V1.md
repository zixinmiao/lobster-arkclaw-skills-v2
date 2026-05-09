# V1 → V2 迁移说明

本文档列出 v2 相对 v1 的全部增量改动，便于既有 v1 调用方平滑迁移。

## v2.1 增量（2026-05-09 更新）

实战中发现两个 agent 用同一套 v2 skill 会创建出字段不一致的 Base（字段集差异 / 类型差异 / 同义字段如 `guidename` vs `guide_name` / 缺表）。根因是 schema 用自然语言 markdown 描述，每个 agent 解读不同。v2.1 收敛：

- **新增 [`references/schema.json`](references/schema.json)**：机器可读权威 schema；所有字段定义、类型、enum、必填、命名禁止表都在这里。bootstrap 必须读 schema.json 直接生成 field-create 命令，不再凭文档自然语言解读。
- **bootstrap 改 schema-driven + 回读校验**：建完字段后必须 `+field-list` 回读比对 schema.json；类型不一致或字段缺失时 `binding_status` 不得置为 `ready`。
- **Canonical 字段命名强约束**：`schema.json.field_naming_rules.forbidden_synonyms` 列出禁止变体（如 `guidename` / `guideName` / `导购姓名` 不得替代 `guide_name`）。
- **删除 `try_on_result` 字段**：试穿结果改由反馈四字段 + `not_buy_reason` + `followup_intent` 自然表达，主表不再单独维护强枚举字段。主表 21→20 字段，required 8→7。

## 1. Skill 列表变化

| v1 Skill | v2 状态 | 说明 |
|---|---|---|
| lobster-fitting-bundle-router | 合并 | 与 record-merge 一并合入 `lobster-fitting-input-pipeline` |
| lobster-fitting-record-merge | 合并 | 与 bundle-router 一并合入 `lobster-fitting-input-pipeline` |
| lobster-fitting-draft-manager | 保留 | 状态机价值高，不动 |
| lobster-tag-ocr | 保留 | 增加 `fabric` / `category` 字段输出 |
| lobster-fitting-stt | 改名 | 改为 `lobster-fitting-voice-extract`，职责定义对齐"transcript 文本结构化"实际能力 |
| lobster-fitting-photo-archive | 删除 | 价值低；图片 URL 直接挂 `media_urls` 进主表 |
| lobster-fitting-bitable-bootstrap | 增强 | 步骤 6「持久化要求」展开为完整 binding 协议 |
| lobster-fitting-bitable-sync | 更新 | 字段列表按 v2 contract 更新 |
| lobster-fitting-card-fill | 更新 | 字段列表按 v2 contract 更新 |
| lobster-member-identity-parse | 保留 | 不动 |
| lobster-followup-lead-sync | 更新 | required 字段从 14 收敛到 8 |
| lobster-member-profile-sync | 更新 | 合并"风格偏好"+"版型偏好"→"穿着偏好"；删除"搭配需求" |
| lobster-followup-reminder-dispatch | 保留 | 接收回访表的"是否已提醒/提醒时间"字段（这两个字段从回访表迁出，归此 skill 维护提醒派发表） |
| lobster-fitting-query | 保留 | 字段引用按新 contract 更新 |
| lobster-fitting-daily-summary | 保留 | 字段引用按新 contract 更新 |
| lobster-script-generate | 删除 | 与试衣录入闭环关系弱，可剥离到独立 agent |
| **lobster-fitting-input-pipeline** | 新增 | 合并 bundle-router + record-merge |
| **lobster-voice-transcribe** | 新增 | 定义 ASR 协议，由 connector 层实现 |
| **lobster-input-validator** | 新增 | 统一前置 required 字段校验 |
| **lobster-tag-ocr-retry** | 新增 | OCR 低置信度兜底交互 |

## 2. 主表「试衣商品记录」字段变化

### 删除（5 个）
- `source_bundle_id`：与 `operator_id` 99% 重叠，运维 trace 即可
- `source_message_ids`：运维 trace，不进业务表
- `source_channel`：当前只有飞书一条入口
- `session_status`：状态归 Session 索引表，主表是事实记录
- `ocr_confidence`：留在素材索引表

### 合并（2 → 1）
- `fitting_date` + `fitting_time` → `fitting_at`（datetime 单字段）

### 简化（1）
- 去掉 `is_member`：用 `member_mobile_last4` 非空判断会员身份

### 新增（2）
- `fabric`：面料
- `category`：品类（外套 / 裤 / 裙 / 配饰）

### 删除（v2.1）
- `try_on_result`：试穿结果改由反馈四字段 + `not_buy_reason` + `followup_intent` 自然表达，主表不再维护此字段

### 保留（不动）
反馈类字段全部保留，由 AI 主动做语义拆分：
- `body_effect_desc`（上身效果）
- `fit_feedback`（版型反馈）
- `liked_points`（喜欢点）
- `disliked_points`（不喜欢点）

### v2.1 主表 required（7 个）
- `session_id`
- `guide_name`
- `fitting_at`
- `operator_id`
- `product_code`
- `product_name`
- `raw_notes`

## 3. 「线索回访表」字段变化

### 迁出（2 个 → 提醒派发表）
- `是否已提醒` `提醒时间` → 归 `lobster-followup-reminder-dispatch` 维护

### required 收敛（14 → 8）
- 仍 required：`线索ID` / `来源试衣记录ID` / `日期` / `导购open_id` / `会员尾号` / `试穿单品` / `回访原因` / `状态`
- 改 recommended：`建议触达时机类型` / `计划触达时间` / `推荐动作` / `优先级` / `建议回访内容` / `导购名称`

## 4. 「会员画像表」字段变化

### 合并（2 → 1）
- `风格偏好` + `版型偏好` → `穿着偏好`

### 删除（1）
- `搭配需求`：可由 `liked_points` 二次衍生

### 保留 required
- `会员尾号` / `最近更新时间` / `最近来源试衣记录` / `画像置信度`

## 5. 新增协议

### `references/binding-config-protocol.md`
明确项目级 binding 的持久化方式：
- 首选 OpenClaw 平台 KV
- 兜底：飞书 Base 内 `_系统配置` 表
- 规定 binding 数据结构、读写时序、`last_verified_at` 校验

### `lobster-voice-transcribe/SKILL.md`
明确 ASR 协议：
- 输入：`audio_file_key` / `audio_url`
- 输出：`transcript` / `asr_confidence` / `asr_engine` / `failure_reason`
- 实际 ASR 调用由 connector 层实现，skill 层只定义合约

## 6. 调用方需要做的改动

1. 把对 `bundle-router` 与 `record-merge` 的两次调用，合并为一次 `input-pipeline` 调用
2. 语音消息进入流水线前，由 connector 调用 `voice-transcribe` 转写为 transcript，再当作文本消息注入 `messages[].text`
3. 写表前，先调用 `input-validator` 做统一字段校验
4. binding 持久化按 `binding-config-protocol.md` 实现
5. 表结构按新 contract 调整字段
