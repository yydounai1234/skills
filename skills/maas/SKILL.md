---
name: maas
description: >
  通过七牛云 MaaS 平台 API 完成 API Key 管理、用量查询、账单查询、请求日志分析、
  模型市场浏览等任务。所有接口均使用 AK/SK 签名鉴权，基础服务地址为
  https://api.qiniu.com/api。
type: tool
best_for:
  - "管理七牛云 MaaS 平台的 API Key（创建、删除、启用/禁用、改名、设限额）"
  - "查询 AI 模型调用的用量数据和账单（按月、按时间范围、按 API Key 分组）"
  - "分页查询请求日志（支持按模型、状态码、API Key 过滤）"
  - "浏览模型市场，获取模型的定价、能力、约束参数等信息"
  - "获取计费项映射，将 key 映射为可读的计费项名称"
scenarios:
  - "用户想创建一个新的 API Key 用于生产环境调用"
  - "用户想查看上个月所有 API Key 的费用和用量分布"
  - "用户想查某个 API Key 本月消耗了多少 token"
  - "用户想禁用一个不再使用的 API Key"
  - "用户想查近 7 天按天粒度的用量趋势"
  - "用户想查询某次失败请求的详情和错误原因"
  - "用户想知道某个模型的价格和支持的参数"
  - "用户想给某个 API Key 设置每日 token 用量上限"
---

# 七牛云 MaaS 平台 API 使用指南

通过调用七牛云 MaaS API，帮助用户管理 API Key、查询用量账单、分析请求日志、浏览模型市场。

## 鉴权方式

所有接口均使用七牛 AK/SK 签名（`QiniuBearerAuth`）进行鉴权。

**Authorization Header 格式：**
```
Authorization: Bearer Qiniu <AccessKey>:<EncodedSign>
```

**签名算法（以 Node.js 为例）：**
```javascript
const crypto = require('crypto');

function sign(accessKey, secretKey, method, path, host, body, contentType) {
  const bodyHash = (body && contentType !== 'application/octet-stream')
    ? crypto.createHash('sha256').update(body).digest('hex')
    : '';

  const signingStr = [method, path, `Host: ${host}`, `Content-Type: ${contentType}`, '', bodyHash, ''].join('\n');
  const sign = crypto.createHmac('sha256', secretKey).update(signingStr).digest('base64url');
  return `Bearer Qiniu ${accessKey}:${sign}`;
}
```

> 详细签名规范及多语言示例请参考七牛官方文档。

---

## 基础信息

| 项目 | 值 |
|------|----|
| 服务域名 | `https://api.qiniu.com/api` |
| 模型市场（国内） | `https://api.qnaigc.com` |
| 模型市场（全球） | `https://openai.sufy.com` |
| 鉴权方式 | Bearer Qiniu AK/SK 签名 |

---

## API 速查

### 1. API Key 管理

#### 创建 API Key
```
POST /inapi/v2/apikey
```
**请求体：**
```json
{ "name": "生产环境 Key" }
```
- `name`：名称标签，长度 1-20 字符
- `type`（可选）：传入 `"member"` 创建 VIP 订阅专用 Key

**响应：** 返回完整的 `key` 值（`sk-` 开头），**仅此一次完整返回，请提醒用户保存**。

---

#### 获取所有 API Key 列表（含配额用量）
```
GET /inapi/v3/apikeys
```
**响应字段：**
- `key`：脱敏后的 Key 值
- `enabled`：是否启用
- `totalTokens`：累计消耗 token 总量
- `quota.daily`/`quota.monthly`/`quota.total`：日/月/总配额（`used`/`limit`）

---

#### 启用或禁用 API Key
```
PUT /inapi/v2/apikey/enabled
```
**请求体：**
```json
{ "key": "sk-xxx", "enabled": false }
```
- **禁用立即生效**，使用该 Key 的请求将被拒绝
- 已禁用的 Key 可重新启用，历史数据不丢失

---

#### 修改 API Key 名称
```
PUT /inapi/v2/apikey/name
```
**请求体：**
```json
{ "key": "sk-xxx", "name": "新名称" }
```

---

#### 设置 API Key 用量限额
```
PUT /inapi/v2/apikey/quota/{api_key}
```
**请求体示例：**
```json
{
  "daily_quota": {
    "enabled": true,
    "limit": 100000,
    "alert_threshold": 80,
    "suppress_alert": false
  },
  "monthly_quota": {
    "enabled": true,
    "limit": 2000000,
    "alert_threshold": 90,
    "suppress_alert": false
  }
}
```
- `limit`：token 数量上限
- `alert_threshold`：告警阈值（百分比 0-100）
- 日限额按 UTC+8 自然日重置，月限额按 UTC+8 自然月重置

---

#### 删除 API Key
```
DELETE /inapi/v2/apikey
```
**请求体：**
```json
{ "key": "sk-xxx" }
```
> **注意：只能删除已禁用的 Key。** 若 Key 当前为启用状态，需先调用禁用接口。删除不可逆，历史账单数据仍会保留。

---

### 2. 账单查询

#### 查询单月账单
```
GET /inapi/v3/stat/bill?month=2025-11&api_key=sk-xxx
```
| 参数 | 说明 |
|------|------|
| `month` | 必填，格式 `YYYY-MM` |
| `api_key` | 可选，不传则返回所有 Key 汇总 |

**响应结构：**
```json
{
  "models": [
    {
      "model_id": "gpt-4",
      "items": [
        { "name": "文本输入 tokens", "usage": { "count": 150.5, "unit": "k/tokens" }, "fee": 4.52, "key": "input" }
      ],
      "total_fee": 6.03,
      "total_requests": 1200
    }
  ]
}
```

---

#### 查询单月所有 API Key 的账单
```
GET /inapi/v3/stat/bill/all_keys?month=2025-11
```
返回所有有数据的 Key 分别的账单，最后一项 `api_key: "all"` 为汇总数据。

---

#### 按时间范围查询账单（多粒度）
```
GET /inapi/v3/stat/bill/range?start=...&end=...&grain=day&api_key=sk-xxx
```
| 参数 | 说明 |
|------|------|
| `start` / `end` | RFC3339 格式，含时区，如 `2025-11-01T00:00:00+08:00` |
| `grain` | `month` / `day` / `hour` / `five_minute` / `minute` |
| `api_key` | 可选 |

**时间范围限制：**
| 粒度 | 最大范围 |
|------|----------|
| month / day | 35 天 |
| hour | 7 天 |
| five_minute / minute | 1 天 |

---

#### 按时间范围查询所有 Key 的账单
```
GET /inapi/v3/stat/bill/range/all_keys?start=...&end=...&grain=day
```

---

### 3. 用量查询（新版）

```
GET /inapi/v3/stat/new?start=...&end=...&g=day&api_key=sk-xxx
```
| 参数 | 说明 |
|------|------|
| `start` / `end` | RFC3339 格式 |
| `g` | 时间粒度：`month` / `day` / `hour` / `five_minute` / `minute` |
| `api_key` | 可选，不传返回全部汇总 |

---

### 4. 请求日志

#### 分页查询请求日志
```
GET /inapi/v3/stat/log?page=1&page_size=20&start=...&end=...
```
| 参数 | 说明 |
|------|------|
| `page` / `page_size` | 必填，page 从 1 开始，page_size 上限 100 |
| `start` / `end` | 可选，RFC3339 格式，范围不超过 35 天 |
| `model` | 可选，精确匹配模型名称 |
| `code` | 可选，按 HTTP 状态码过滤（优先级高于 `status`） |
| `status` | 可选，`success` / `failure` / `client_error` / `server_error` |
| `apikey` | 可选，精确匹配 API Key（不含 Bearer 前缀） |
| `server_type` | 可选，`chat`（默认）/ `image` / `video` |
| `id` | 可选，精确查询某条日志 ID |

**响应字段（每条日志）：**
- `id`：请求 ID（前缀 `chatcmpl-` / `chatimage-` / `qvideo-`）
- `model_id`：调用的模型
- `api_key`：使用的 Key（脱敏）
- `start_time` / `end_time`：请求时间段
- `code`：HTTP 状态码
- `state`：`success` / `fail`
- `usage`：用量（`input`/`output` 等 token 数）
- `bo_usage`：按计费项 key 的用量，配合 `/inapi/v3/market/pricingitems` 查看名称

---

#### 查询单条日志详情
```
GET /inapi/v3/stat/log/detail?request_id=chatcmpl-abc123
```
根据 `request_id` 前缀自动路由：
- `chatcmpl-` → 对话日志
- `chatimage-` → 图片任务
- `qvideo-` → 视频任务

**额外返回字段：**
- `cost_time`：各阶段耗时（`ttft` 首字耗时、`latency` 总耗时，单位毫秒）
- `user`：请求来源（`uid`、`client_ip`、`user_agent`）

---

### 5. 模型市场

#### 获取模型列表
```
GET /v1/market/models?sort=rank&order=desc
```
> **注意：** 此接口需使用**服务域名**直接访问，不经过 `/api` 前缀：
> - 国内：`https://api.qnaigc.com/v1/market/models`
> - 全球：`https://openai.sufy.com/v1/market/models`（支持 `overseas=true`）

**响应包含每个模型的：**
- `id` / `name` / `description`：基础标识
- `model_constraints`：上下文长度、最大输出 token 数等
- `architecture`：支持的输入/输出模态，以及 function calling、reasoning、content_cache 等能力
- `pricing_rules_v2`：按 `details_v2` 字段查看各计费项（`input`/`output`/`cache` 等）单价
- `rate_limit`：RPM / TPM 等限流配置

---

#### 获取计费项列表
```
GET /inapi/v3/market/pricingitems
```
返回平台所有计费项的 key → 名称映射，用于将账单/日志中的 `key` 字段转换为可读名称（如 `input` → "文本输入 tokens"）。

---

## 常见任务流程

### 查询上月费用
1. `GET /inapi/v3/stat/bill?month=YYYY-MM` 获取汇总账单
2. 若需按 Key 细分：`GET /inapi/v3/stat/bill/all_keys?month=YYYY-MM`
3. 若需知道计费项名称：先 `GET /inapi/v3/market/pricingitems` 获取映射

### 新建并限制 API Key
1. `POST /inapi/v2/apikey` 创建 Key，**提醒用户保存完整 key 值**
2. `PUT /inapi/v2/apikey/quota/{api_key}` 设置日/月限额

### 排查失败请求
1. `GET /inapi/v3/stat/log?status=failure&page=1&page_size=20` 查询失败日志列表
2. `GET /inapi/v3/stat/log/detail?request_id=<id>` 查看某次请求的详细信息和耗时

### 下线一个 API Key
1. `PUT /inapi/v2/apikey/enabled`（`enabled: false`）禁用
2. 确认无影响后：`DELETE /inapi/v2/apikey` 删除

---

## 注意事项

- **API Key 完整值仅在创建时返回一次**，后续接口均返回脱敏版本，务必提醒用户创建后立即保存
- **删除只能针对已禁用的 Key**，否则接口会返回错误
- **禁用操作立即生效**，生产环境操作前确认业务侧已停止使用
- 账单接口目前无需鉴权（`security: []`），但日志和 Key 管理接口需要鉴权
- `bo_usage` 字段的 key 需配合 `/inapi/v3/market/pricingitems` 接口解析为可读名称
