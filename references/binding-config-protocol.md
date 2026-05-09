# Binding Config Protocol (v2 新增)

本文件定义"导购小龙虾"项目级 binding 的持久化协议，规定 binding 数据结构、读写时序、两层兜底机制。v1 的 `lobster-fitting-bitable-bootstrap` 步骤 6「持久化要求」在 v2 中展开为本协议。

## 为什么要明确这个协议

v1 中"项目级共享 binding"是口头约定，没有规定 binding 存到哪里。这意味着：
- 第二个导购进入时是否真的能复用第一个导购建的表，依赖 connector / runtime 的具体实现
- skill 层无法保证"多导购共用同一套 Base"的承诺真的落地

v2 把这件事协议化，给 connector 实现方一个明确合约。

## 数据结构

binding 对象的完整 schema：

```json
{
  "schema_version": "v2",
  "binding_scope": "project",
  "project_id": "lobster-fitting-shop-001",
  "base_token": "bascn_xxx",
  "base_url": "https://xxx.feishu.cn/base/bascn_xxx",
  "tables": {
    "试衣商品记录": "tbl_xxx",
    "试衣Session索引": "tbl_xxx",
    "试衣素材索引": "tbl_xxx",
    "线索回访表": "tbl_xxx",
    "会员画像表": "tbl_xxx",
    "提醒派发表": "tbl_xxx",
    "_系统配置": "tbl_xxx"
  },
  "field_map": {
    "试衣商品记录": {
      "session_id": "fld_xxx",
      "guide_name": "fld_xxx",
      "fitting_at": "fld_xxx",
      "operator_id": "fld_xxx",
      "product_code": "fld_xxx",
      "product_name": "fld_xxx",
      "try_on_result": "fld_xxx",
      "raw_notes": "fld_xxx"
    }
  },
  "created_at": "2026-05-09T10:00:00+08:00",
  "created_by_operator_id": "ou_xxx",
  "last_verified_at": "2026-05-09T10:00:00+08:00",
  "binding_status": "ready"
}
```

### 字段说明

| 字段 | 必需 | 说明 |
|---|---|---|
| `schema_version` | required | 协议版本，当前 `v2` |
| `binding_scope` | required | 默认 `project`；预留 `tenant` / `store` 等更高粒度 |
| `project_id` | required | 项目唯一标识，KV 存储的 key |
| `base_token` | required | 飞书 Base token |
| `base_url` | required | 飞书 Base URL，用于人工排查 |
| `tables` | required | 表名 → table_id 映射；至少包含 6 张业务表 + `_系统配置` |
| `field_map` | recommended | 表 → 字段名 → field_id 二级映射；可选但强烈建议存，避免每次写入前重新读字段 |
| `created_at` | required | 首次创建时间 |
| `created_by_operator_id` | recommended | 首次创建人 open_id，用于追溯 |
| `last_verified_at` | required | 最近一次 schema 校验通过时间 |
| `binding_status` | required | `ready` / `partial` / `missing` / `invalid` |

## 两层兜底持久化

### 第一层（首选）：OpenClaw 平台 KV 存储

```
KV.put(key=`binding:${project_id}`, value=<binding JSON>)
KV.get(key=`binding:${project_id}`)
```

要求：
- 平台 KV 应在项目级生效，不绑定到具体 sender / session
- TTL：建议无 TTL 或 ≥ 30 天；写入新 binding 时主动覆盖
- 可用性应高于飞书 Base 本身

### 第二层（兜底）：飞书 Base 内 `_系统配置` 表

当平台 KV 不可用、或 connector 启动时只知道 `base_token` 但 KV miss 时，从 Base 内自举读取。

`_系统配置` 表结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `config_key` | 文本 | 配置键，binding 用 `binding:project` |
| `config_value` | 长文本 | binding JSON 字符串 |
| `schema_version` | 文本 | `v2` |
| `updated_at` | 日期时间 | 更新时间 |
| `updated_by_operator_id` | 文本 | 更新人 |

写入时，KV 与 `_系统配置` 表**双写**；读取时优先 KV，失败回落 `_系统配置`。

## 读写时序

### 导购 A 首次进入项目

```text
1. read KV[`binding:${project_id}`]
   → miss
2. 提示导购给链接 / 走 bootstrap 建表
3. 解析链接 / 创建 Base 与 7 张表（含 _系统配置）
4. 读取真实字段，构建 field_map
5. 校验 schema 完整性 → binding_status = ready
6. 双写：
     KV.put(`binding:${project_id}`, binding)
     _系统配置.upsert(config_key=`binding:project`, config_value=JSON.stringify(binding))
7. 返回 binding 给上游 sync skill
8. 写入主表
```

### 导购 B 后续进入

```text
1. read KV[`binding:${project_id}`] → hit
2. 检查 last_verified_at
   - 距今 < 24h → 直接复用 binding，进入 sync
   - 距今 ≥ 24h → 触发轻量校验：调用飞书 list-fields 比对 field_map
     - 一致 → 更新 last_verified_at，写入 KV，进入 sync
     - 不一致 → 进入 bootstrap 修复流程
```

### 平台 KV 不可用时

```text
1. read KV → fail
2. 若 connector 已知 base_token（从环境变量 / 项目配置）：
   read _系统配置.where(config_key=`binding:project`)
   → 拿到 binding → 反向写回 KV（best effort）
3. 若 connector 不知道 base_token：
   要求人工提供 Base 链接，触发 bootstrap 重新建立 binding
```

## 失效与刷新

### `last_verified_at` 阈值
- 默认 24 小时
- 写表 skill 在写入前检查；超时则触发轻量字段校验
- 轻量校验只检查 required 字段是否还在；不重读全部字段类型

### 主动失效场景
以下情况必须主动刷新 binding：
- bootstrap 检测到字段缺失或表不存在
- sync 写入时返回飞书错误（如 `field_not_found`）
- 人工触发"重新建表"动作

### 刷新动作
```
1. 重新读取真实表结构 + 字段
2. 修复缺失（建表 / 补字段）
3. 更新 binding.field_map / binding.tables
4. 更新 last_verified_at
5. 双写 KV + _系统配置
```

## 并发与冲突

- 同一 `project_id` 下，所有写入操作建议串行（队列化）
- 多导购同时进入"首次建表"时，应用分布式锁（如 KV.set with NX）；获得锁的实例执行 bootstrap，其他实例等待并复用结果
- binding 不应按 sender / session 分裂；否则违背"项目级共享"承诺

## 安全边界

- binding 中只存表/字段元数据，不存业务数据
- `_系统配置` 表的访问权限应与业务表对齐（同一个 Base 内的应用授权）
- binding 不应作为对外 API 暴露
- 调试时打印 binding 应脱敏 `created_by_operator_id`

## skill 层的责任

| skill | 责任 |
|---|---|
| `lobster-fitting-bitable-bootstrap` | binding 创建、修复、双写 |
| `lobster-fitting-bitable-sync` | 读取 binding、检查 `last_verified_at`、写入主表 |
| `lobster-followup-lead-sync` / `lobster-member-profile-sync` | 同上 |
| connector / runtime | 提供 KV 实现；提供 `_系统配置` 表读写能力；管理分布式锁 |

## 兼容性

- v1 的 binding 对象（无 `field_map` / `last_verified_at`）应被识别为 `schema_version=v1`，触发一次完整校验后升级为 v2
- v2 工具不应静默兼容残缺 binding，至少要 log 出兼容路径
