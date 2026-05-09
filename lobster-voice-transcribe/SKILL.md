---
name: lobster-voice-transcribe
description: 把飞书消息中的音频附件转写为 transcript 文本。本 skill 定义 ASR 协议合约（输入/输出/失败语义），实际 ASR 调用由 connector 层实现（飞书开放平台语音识别 / 火山引擎 ASR / 阿里云 ASR / Whisper 等）。适用于试衣场景导购发送语音消息时，把音频统一转为文本后再走 input-pipeline。
---

# 语音转写（ASR 协议层，v2 新增）

## 为什么独立成 skill

v1 的 `lobster-fitting-stt` 实际只处理"已经拿到 transcript"的内容，**没有真正的音频→文本能力**。这导致语音消息进来只能等导购自己打字补充。

v2 把"音频→文本"从 skill 层下沉到 connector 层，但保留一个明确的协议合约 skill，便于：
- 给 connector 实现方一个标准合约（输入字段、输出字段、失败语义）
- 让上游编排器（workflow / runtime）知道在哪一步注入 ASR 调用
- 把 transcript 注入 `messages[].text` 后，与文本消息走同一条 input-pipeline

## 调用时机

在 input-pipeline 之前。当 connector 检测到消息类型是音频时，先调用本 skill，再把 transcript 注入文本通道。

```
飞书音频消息
  ↓ (connector 检测 message.msg_type = audio)
lobster-voice-transcribe (本 skill)
  ↓ (transcript / failure_reason)
注入 messages[].text 或 voice_extract_result
  ↓
lobster-fitting-input-pipeline
```

## 输入

| 字段 | 必需 | 说明 |
|---|---|---|
| `audio_file_key` | 二选一 | 飞书消息附件 file_key |
| `audio_url` | 二选一 | 直接可访问的音频 URL |
| `audio_format` | 推荐 | `mp3` / `m4a` / `wav` / `opus`（飞书默认 opus） |
| `audio_duration_ms` | 推荐 | 音频时长，用于成本控制与超时判断 |
| `language_hint` | 可选 | `zh-CN` / `zh-HK` / `en-US` 等，提升识别率 |
| `domain_hint` | 可选 | 业务领域提示，如 `retail_fitting`，用于偏置词典 |
| `boost_words` | 可选 | 业务热词列表（如品牌名、SKU 前缀），用于纠正常见识别错误 |

## 输出

返回 `result`：

| 字段 | 必需 | 说明 |
|---|---|---|
| `transcript` | 成功时必返 | 转写后的纯文本，已做基本断句 |
| `transcript_segments` | 可选 | 分段转写，含起止时间戳 |
| `asr_confidence` | 推荐 | 整体置信度 0~1 |
| `asr_engine` | 必返 | 实际使用的 ASR 引擎，如 `feishu_asr` / `volc_asr` / `whisper-large-v3` |
| `language_detected` | 可选 | 实际识别的语言 |
| `failure_reason` | 失败时必返 | 见下方失败枚举 |
| `raw_response` | 可选 | 原始响应快照，用于排查 |

## 失败语义（强枚举）

`failure_reason` 必须从以下枚举中选：

| 值 | 含义 | 上游处理 |
|---|---|---|
| `audio_download_failed` | 飞书附件下载失败 | 提示导购重发 |
| `audio_decode_failed` | 音频格式不支持或损坏 | 提示导购换格式 |
| `audio_too_short` | 音频过短（< 1s） | 视为无效消息 |
| `audio_too_long` | 音频过长（超过 ASR 上限） | 提示导购分段发 |
| `transcribe_failed` | ASR 服务返回错误 | 重试 / 降级到备用引擎 |
| `transcribe_timeout` | ASR 调用超时 | 重试 / 降级 |
| `quota_exceeded` | 额度耗尽 | 切换引擎或人工介入 |
| `unknown` | 其他未分类 | 进入 pending_media |

## 执行规则

- **不得伪造 transcript**：失败时必须返回明确 `failure_reason`，不能编造文本兜底
- **不得做业务判断**：本 skill 只负责转写，不做字段抽取（字段抽取由 `lobster-fitting-voice-extract` 负责）
- **数字识别低置信度提示**：transcript 中含 4 位数字（疑似会员尾号、价格、SKU 后缀）时，建议在 `transcript_segments` 中标记为 `low_confidence_numeric`，让下游卡片确认环节由导购纠正
- **业务热词建议**：调用方应传入 `boost_words`（品牌名、常见 SKU 前缀如 `EBF` / `MOC`），显著提升数字串识别率
- **多 ASR 引擎兼容**：connector 实现可串多个引擎（首选飞书原生 → 备用火山 → 兜底 Whisper），但本 skill 只暴露统一协议

## ASR 引擎选择建议（给 connector 实现方）

| 引擎 | 适用 |
|---|---|
| 飞书开放平台语音识别 | 权限链路最短，飞书原生附件可直接转写；推荐首选 |
| 火山引擎 ASR | 字节系，中文识别准确率高，对方言/口音友好 |
| 阿里云 / 腾讯云 ASR | 备选，按企业已有云厂商选 |
| 自建 Whisper | 成本最低，但需自行运维；适合内网部署 |

## 输出格式

成功：
```json
{
  "result": {
    "transcript": "...",
    "transcript_segments": [
      {"start_ms": 0, "end_ms": 2300, "text": "...", "confidence": 0.95, "low_confidence_numeric": false}
    ],
    "asr_confidence": 0.92,
    "asr_engine": "feishu_asr",
    "language_detected": "zh-CN"
  }
}
```

失败：
```json
{
  "result": {
    "failure_reason": "audio_decode_failed",
    "asr_engine": "feishu_asr"
  }
}
```
