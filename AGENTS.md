# AGENTS.md - lobster-arkclaw-skills-v2

本仓库是一套可复用的导购试衣技能包，默认面向"多导购共用同一套飞书试衣台账"的场景。v2 相对 v1 在字段精简、语音处理、binding 持久化三方面做了显式改进。

## 核心职责

本工作区用于处理服饰导购试衣场景，包括：

- 试衣语音转写与文本结构化
- 吊牌 OCR 商品识别（含面料、品类）
- 顾客身份与会员信息解析
- 多消息 bundle 归并、session 判断、字段合并（合并到 input-pipeline）
- 草稿状态管理与卡片确认
- 飞书多维表格写入与查询
- 回访线索生成与提醒派发
- 会员画像沉淀
- 每日汇总

## 飞书多维表格绑定与建表规则

任何写表 skill 在执行前，必须先确保 `bitable_binding` 存在且可用。

### 共享绑定规则
- 默认使用项目级共享 `bitable_binding`
- 同一项目下，所有导购共用同一套飞书多维表格
- 不按导购个人、会话、session 单独创建 Base
- 若系统中已存在可用 binding，则后续导购直接复用，不再重复创建 Base

### 持久化协议（v2 新增明确）
binding 必须按 [`references/binding-config-protocol.md`](references/binding-config-protocol.md) 定义的协议持久化：
- **首选**：OpenClaw 平台级 KV 存储，key = `project_id`
- **兜底**：飞书 Base 内 `_系统配置` 表（meta-table），存放 binding 自举入口
- binding 对象至少包含：`base_token` / `tables` / `field_map` / `last_verified_at`
- `last_verified_at` 超过阈值（默认 24h）后，下次写入前必须重新校验 schema

### 强制流程
1. 若不存在 `bitable_binding.base_token`，先执行"建表初始化"
2. 若 `base_token` 存在但目标表不存在，自动补建缺失表
3. 若表存在但必要字段缺失，自动补齐字段
4. 只有当 `binding_status = ready` 时，才允许进入正式写表
5. 若建表失败，不得伪造写入成功，必须明确返回失败原因

### 默认创建对象
- Base 名称：`导购小龙虾试衣台账`
- 主表：`试衣商品记录`
- 辅助表：`试衣Session索引`
- 辅助表：`试衣素材索引`
- 业务表：`线索回访表`
- 业务表：`会员画像表`
- 系统表：`_系统配置`（v2 新增，用于 binding 自举）

### 执行原则
- 建表与写表分离
- 建表过程必须幂等：重复执行不会创建重复结构
- 优先修复已有 Base，而不是每次新建
- 所有飞书 Base 操作串行执行，避免并发冲突
- 成功建表后，立即按持久化协议持久化 binding 信息

### 主表必备区分字段
主表 `试衣商品记录` 必须保留以下字段（用于多导购共用时的归属区分）：
- `guide_name`
- `store_name`
- `operator_id`（导购 open_id）

## 默认工作原则

1. 先识别"这一条信息属于哪个 session / bundle"
2. 先判断信息是否足够，再决定是否允许正式写入
3. 如果只有图片、没有足够文本，不得直接推断成交结果（保持 v1 强约束）
4. 如果商品主体不明确，优先待补充，不强写主表
5. 如果当前 OCR 商品与历史上下文冲突，优先使用本次吊牌识别结果
6. 默认优先降低串单风险，而不是追求一次写满

## 写入约束

- `pending_media` 状态下，不写正式主表记录
- `write_decision.allow_write = false` 时，不得强行写入
- 禁止编造库存、折扣、活动、到货时间
- 禁止把模糊反馈写成明确成交结论
- `guide_name` 必须来自导购明确自报、已确认 profile、卡片确认或可信员工资料；缺失时必须追问，不得用"导购"、"店员"、"未知"、"默认导购"等占位词填充
- 字段命名严格 canonical（如 `guide_name` / `operator_id` / `fitting_at`），禁止建变体（`guidename` / `guideName` / `导购姓名` 等）；完整禁止表见 `references/schema.json`
- 线上出现多个项目 Base 候选时，不得继续创建新 Base；必须先确认唯一 canonical Base，并把 binding 持久化后再写入

## 多维反馈字段填充原则

主表 `body_effect_desc` / `fit_feedback` / `liked_points` / `disliked_points` 四个字段不依赖导购口语自己区分，而由 AI 基于导购原始输入自行做语义拆分：
- `body_effect_desc`：客户实际穿着后体型呈现（如"显瘦"、"显高"、"肩窄"）
- `fit_feedback`：版型反馈（如"腰部偏紧"、"袖长合适"）
- `liked_points`：客户主观喜欢点（如"喜欢面料"、"颜色好看"）
- `disliked_points`：客户主观不喜欢点（如"颜色偏深"、"版型太宽松"）

不强求每次都写满四个字段，但能拆出哪个写哪个，不要合并到单字段。

## 输出偏好

默认输出应尽量包含：
- 当前识别到的 session / bundle 判断
- 商品候选或主商品（含面料、品类）
- 顾客信息（若已知）
- 试穿反馈摘要（按四个维度分别填充）
- 是否允许写入
- 缺失信息 / 待确认点
- 下一步建议动作

## 安全边界

- 不对外发送未经确认的经营承诺
- 不做超出已知事实的销售话术扩写
- 涉及正式入库、同步、派发提醒时，严格遵守 skill 的字段约束
