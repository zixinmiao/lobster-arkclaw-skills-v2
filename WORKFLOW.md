# WORKFLOW.md - lobster-arkclaw-skills-v2

本文档定义 v2 skill 的推荐编排顺序与强约束。

> 说明：`WORKFLOW.md` 负责描述应该如何串联；真正执行仍需要 runtime、connector 或 workflow engine 来调用这些 skill。

## 目标

这套技能包默认面向以下场景：
- 多导购共用同一套飞书试衣台账
- 首次使用时完成飞书表绑定或初始化
- 试衣记录、回访线索、会员画像稳定写入同一套项目级数据底座
- required 字段缺失时阻断写入，优先保证数据稳定性
- 语音消息可被处理（v2 新增）

## 总体编排原则
1. 先归并、再识别、再合并、再判断是否允许写入
2. 正式写表前，必须先确保 `bitable_binding` 已存在且可用
3. 若没有 binding，优先走"给链接 → 自动登记 → 持久化到协议位置"
4. `pending_media` 或 `write_decision.allow_write = false` 时，不得写入主表
5. required 字段缺失时，不得部分落表（统一交由 `lobster-input-validator` 校验）

## 主流程

```text
message input
  ↓
（音频附件？→ lobster-voice-transcribe → 注入 transcript 到 messages[].text）
  ↓
lobster-fitting-input-pipeline           # 归并 + session 判断 + 字段合并 + pending_media 判定
  ↓
lobster-fitting-draft-manager            # 草稿状态机
  ↓
lobster-tag-ocr / lobster-fitting-voice-extract / lobster-member-identity-parse
（OCR 低置信度 → lobster-tag-ocr-retry → 引导导购重拍）
  ↓
[optional] lobster-fitting-card-fill     # 卡片预填 → 等待导购确认提交
  ↓
lobster-input-validator                  # 写入前最后一道 required 字段校验
  ↓
lobster-fitting-bitable-bootstrap        # 确认 binding ready / 表存在 / 字段完整
  ↓
lobster-fitting-bitable-sync             # 主表写入
  ↓
lobster-followup-lead-sync               # 条件触发：写入回访表
lobster-member-profile-sync              # 条件触发：写入会员画像
lobster-followup-reminder-dispatch       # 条件触发：维护提醒派发表
lobster-fitting-daily-summary            # 异步：每日汇总
```

## 关键前置规则

### 1. 导购资料继承
runtime / connector 应先读取 sender 级资料缓存，至少尝试继承：
- `guide_name`
- `store_name`
- `operator_id`

若同一 sender 之前已经明确提供过导购姓名或门店，后续消息默认继承，不应反复询问。

### 2. 主表 required 字段（v2，8 个）
主表写入前必须具备：
- `session_id`
- `guide_name`
- `fitting_at`
- `operator_id`
- `product_code`
- `product_name`
- `try_on_result`
- `raw_notes`

### 3. operator_id 透传规则
如果来源是飞书消息，runtime / connector 必须把 sender open_id 作为 `operator_id` 直接透传。该字段不依赖模型提取，不依赖导购补充。

> v1 的 `source_bundle_id` 字段在 v2 已删除（与 `operator_id` 重复），统一由 `operator_id` 承担。

### 4. 语音处理边界（v2 改进）
v2 引入显式 ASR 协议：
- 音频消息进入流水线前，由 connector 调用 `lobster-voice-transcribe` 把音频转为 `transcript`
- transcript 注入 `messages[].text` 后，与文本消息走同一条 input-pipeline
- ASR 失败时（`audio_download_failed` / `audio_decode_failed` / `transcribe_failed`），不得伪造 transcript；进入 `pending_media` 等待补充
- transcript 中数字字段（货号、价格、尺码、会员尾号）置信度低，应在卡片确认环节由导购纠正

## 建表 / 初始化规则

正式写表前必须执行：
- `lobster-fitting-bitable-bootstrap`

默认创建 / 校验对象：
- Base：`导购小龙虾试衣台账`
- 主表：`试衣商品记录`
- 辅助表：`试衣Session索引` / `试衣素材索引`
- 业务表：`线索回访表` / `会员画像表`
- 系统表：`_系统配置`（v2 新增，binding 自举入口）

初始化检查时，不仅检查表是否存在，也要检查 required 字段是否完整。

binding 持久化遵循 [`references/binding-config-protocol.md`](references/binding-config-protocol.md)：
- 首选：OpenClaw 平台 KV
- 兜底：飞书 Base 内 `_系统配置` 表
- `last_verified_at` 超过阈值（默认 24h）后强制重新校验

## 写表载荷生成
执行：
- `lobster-fitting-bitable-sync`

目标：
- 把结构化试衣数据映射成适合飞书 Base 写入的 payload
- 在正式写入前完成 required 字段校验（委托给 `lobster-input-validator`）

## 正式写入前必须满足
- `binding_status = ready`
- `write_decision.allow_write = true`
- `validator_result.can_write = true`
- 主表结构完整

禁止写入场景：
- `pending_media`
- `write_decision.allow_write = false`
- binding 不可用
- required 字段缺失

## 后置沉淀

成功写表后，按条件继续执行：
- `lobster-followup-lead-sync`：当试穿反馈呈现"高意向 + 未成交 + 原因明确 + 是会员"时触发
- `lobster-member-profile-sync`：当 `member_mobile_last4` 非空时触发
- `lobster-followup-reminder-dispatch`：根据回访线索的"建议触达时机"派发提醒
- `lobster-fitting-daily-summary`：异步汇总

### 回访表 required 字段（v2，8 个）
- `线索ID`
- `来源试衣记录ID`
- `日期`
- `导购open_id`
- `会员尾号`
- `试穿单品`
- `回访原因`
- `状态`

### 会员画像表 required 字段
- `会员尾号`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

## 推荐前置判断（伪代码）

```text
if message has audio:
  transcript = lobster-voice-transcribe(audio)
  if transcript.failure_reason: enter pending_media
  else: messages[].text += transcript

if no project-level bitable_binding:
  if table/base link is provided:
    parse link → validate fields → save binding (per binding-config-protocol)
  else:
    run bootstrap fallback

if binding_status != ready:
  stop write

if write_decision.allow_write != true:
  stop write

validator_result = lobster-input-validator(record, target_table)
if validator_result.can_write != true:
  stop write

run lobster-fitting-bitable-sync
run base writer
```
