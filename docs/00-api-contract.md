# 00 · 接口契约（并行开发唯一事实来源）

> 任何模块改动接口，必须先改本文件。金额单位一律 **微元（micro-yuan）**：`1 元 = 1_000_000`。
> 所有内部 JS 使用 camelCase，数据库列使用 snake_case，转换在路由层完成。

## 1. 目录与文件归属

| 路径 | 归属 | 说明 |
|---|---|---|
| `src/config.js` `src/db.js` `src/server.js` | LEAD | 已完成，禁止改动 |
| `src/util/*` `src/http/*` `src/middleware/*` `src/security/*` `src/obs/*` | LEAD | 已完成，禁止改动 |
| `src/billing/pricing.js` `src/billing/quota.js` | LEAD | 已完成，禁止改动 |
| `src/upstream/index.js` `src/upstream/dispatch.js` `src/routes/v1.js` | LEAD | 已完成，禁止改动 |
| `src/routes/public.js` | LEAD | 已完成，禁止改动 |
| `src/upstream/anthropic.js` `src/upstream/gemini.js` | A | 新增 |
| `src/routes/user.js` `src/routes/admin.js` `src/routes/payment.js` | B | 新增 |
| `src/services/*.js` | B | 新增 |
| `public/**` | C | 新增/替换 |
| `deploy/**` `docs/01~07` `README.md` `scripts/**` | D | 新增 |
| `tests/**` | E | 新增 |

## 2. 核心模块可调用 API（只读引用）

### `src/db.js`
```js
initDb() -> DatabaseSync        // server 启动已调用
get(sql, ...params) -> row|null
all(sql, ...params) -> row[]
run(sql, ...params) -> {changes, lastInsertRowid}
tx(fn) -> fn 返回值             // BEGIN IMMEDIATE / COMMIT / ROLLBACK
getSetting(key, fallback), getSettingInt(key, fallback), setSetting(key, value), allSettings()
addBalance(userId, deltaMicro, kind, ref, note) -> 新余额
audit(actorUserId, action, target, detailObject, ip)
```
表结构见 `src/db.js` 顶部 `SCHEMA`，字段以那里为准。

### `src/billing/pricing.js`
```js
getModelPrice(model) -> price|null
requireModelPrice(model) -> price      // 未开通/下线直接抛 ApiError
calcCharge(price, promptTokens, completionTokens) -> {chargeMicro, costMicro, marginMicro}
estimateRequestTokens(body) -> number
estimateReserveMicro(price, body) -> number
listModels({onlyEnabled}) -> [{model, displayName, vendor, inputPriceMicro, outputPriceMicro,
                               upstreamInMicro, upstreamOutMicro, perRequestMicro, enabled, note}]
upsertModelPrice(input)
```

### `src/billing/quota.js`
```js
reserveQuota({userId, tokenId, reserveMicro}) -> reservation   // 余额不足抛 402
settleQuota({reservation, price, promptTokens, completionTokens, meta}) -> {chargeMicro, costMicro, refundMicro}
refundQuota({reservation, meta})                                // 失败全额退
defaultRpm() -> number
defaultConcurrent() -> number
```

### `src/middleware/auth.js`
```js
parseCookies(req) -> {name:value}
extractApiKey(req) -> string|null          // Bearer / x-api-key
authenticateApiToken(rawKey) -> {apiToken, user}
createSession(userId, {ip, userAgent}) -> sid
sessionCookieHeader(sid) / clearSessionCookieHeader()
destroySession(sid)
userFromRequest(req) -> user|null          // server 已注入 ctx.user
requireUser(ctx) -> user                   // 未登录抛 401
requireAdmin(ctx) -> user                  // 非 admin 抛 403
SESSION_COOKIE = 'rs_sid'
```

### `src/util/crypto.js`
```js
sha256Hex, hmacSha256Hex, randomString, timingSafeEqualStr
hashPassword(pw) / verifyPassword(pw, stored)
encryptSecret / decryptSecret / maskSecret
generateApiToken() -> {plain, hash, prefix, suffix}
generateCode(prefix, groups, groupLen)
orderNo(prefix)
```

### `src/util/errors.js`
```js
new ApiError(status, message, {code, type, param, headers})
badRequest(msg, param) / unauthorized() / forbidden() / notFound() / conflict() / insufficientQuota()
```

### `src/http/respond.js`
```js
json(res, status, payload, headers?)
text(res, status, body, contentType?, headers?)
noContent(res) / redirect(res, location)
applyCors(res)                          // server 已调用
```

### `src/obs/log.js`
```js
log.info/warn/error/debug(msg, fields)
metrics.snapshot()
```

## 3. 路由注册约定

每个 `src/routes/*.js` 必须导出：
```js
export function register(router) { router.get('/api/xxx', handler); }
```
- `handler(ctx)` 返回值忽略；**自己负责 `json()/text()` 输出**。
- `ctx = {req, res, url, method, path, query, params, ip, body, user, routeMeta}`。
- `ctx.user` 可能为 null（未登录）。需要登录就调 `requireUser(ctx)`。
- 抛 `ApiError` 即可，`server.js` 统一转 JSON 错误体。
- 路径写全（含 `/api` 前缀），模块之间不要互相 import 路由文件。

## 4. HTTP 接口清单

### 4.1 公开（已实现：`public.js`）
| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/healthz` `/metrics` | 存活/指标（server.js） |
| GET | `/api/site` | 站点名、公告、注册开关、最低充值 |
| GET | `/api/pricing` | 对客价目（不含成本） |
| GET | `/api/models` | 模型列表 |

### 4.2 认证与会话（B · `user.js`）

**登录必须接爆破防护**（`src/security/ratelimit.js`）：
```js
assertLoginAllowed(ctx.ip, email)     // 开头调用，锁定则自动抛 429 login_locked
recordLoginFailure(ctx.ip, email)     // 密码/邮箱错误分支
clearLoginFailures(ctx.ip, email)     // 登录成功后
assertRpm(`reg:${ctx.ip}`, 10)        // 注册接口按 IP 限流
```

| 方法 | 路径 | 入参 | 返回 |
|---|---|---|---|
| POST | `/api/auth/register` | `{email,password,inviteCode?}` | `{user}` + Set-Cookie |
| POST | `/api/auth/login` | `{email,password}` | `{user}` + Set-Cookie |
| POST | `/api/auth/logout` | — | `{ok:true}` + 清 Cookie |
| GET | `/api/auth/me` | — | `{user,site}` |
| POST | `/api/auth/password` | `{oldPassword,newPassword}` | `{ok:true}` |

`user` 结构：`{id,email,displayName,role,balanceMicro,inviteCode,status,createdAt}`（**永不含 password_hash**）。

### 4.3 用户控制台（B · `user.js`）
| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/keys` | 我的 Key 列表（含 `remainingMicro`，不含 hash） |
| POST | `/api/keys` | `{name,quotaMicro?,rpmLimit?,concurrentLimit?,expiresInDays?}` → 返回一次 `plainToken` |
| PATCH | `/api/keys/:id` | 改名/额度/限速/启停 |
| DELETE | `/api/keys/:id` | 删除 |
| GET | `/api/usage?page=&pageSize=&from=&to=&model=` | 我的调用明细 + `summary` |
| GET | `/api/usage/summary?days=30` | 每日曲线 + 按模型汇总 |
| GET | `/api/ledger?page=` | 资金流水 |
| GET | `/api/orders?page=` | 我的充值订单 |
| POST | `/api/redeem` | `{code}` → 卡密充值 |
| POST | `/api/recharge` | `{amountMicro,payType}` → `{orderNo,payUrl,qrText}`（内部调 `payment.js` 导出的 `createOrder`） |
| GET | `/api/dashboard` | 控制台首屏汇总 |

### 4.4 管理后台（B · `admin.js`，全部要求 `requireAdmin`）

> `requireAdmin(ctx)` 已支持运维服务令牌：请求头 `Authorization: Bearer <RELAY_ADMIN_SERVICE_TOKEN>`
> 时 `ctx.serviceAdmin=true`，返回的 actor 为 `{id:0, role:'admin'}`。审计日志会记 actor=0，属正常。
| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/admin/overview` | 用户数/今日收入/今日成本/毛利/请求量/余额负债 |
| GET | `/api/admin/users?q=&page=` | 用户列表 |
| POST | `/api/admin/users` | 新建用户 |
| GET | `/api/admin/users/:id` | 用户详情（钥匙、用量、流水） |
| PATCH | `/api/admin/users/:id` | 状态/角色/备注 |
| POST | `/api/admin/users/:id/balance` | `{deltaMicro,note}` 手工加减款（写 ledger + audit） |
| GET | `/api/admin/channels` | 渠道列表（Key 掩码） |
| POST | `/api/admin/channels` | 新建渠道 |
| PATCH | `/api/admin/channels/:id` | 修改 || DELETE | `/api/admin/channels/:id` | 删除 |
| POST | `/api/admin/channels/:id/test` | `{model}` 实测连通性 |
| POST | `/api/admin/channels/:id/reset` | 健康分复位 |
| GET | `/api/admin/models` | 含成本价的模型表 |
| POST | `/api/admin/models` | 新增/改价 |
| DELETE | `/api/admin/models/:model` | 删除 |
| GET | `/api/admin/usage?...` | 全站调用日志 |
| GET | `/api/admin/orders?status=&page=` | 订单 |
| POST | `/api/admin/orders/:id/mark-paid` | 手工补单 |
| GET/POST | `/api/admin/codes` | 卡密列表 / 批量生成 `{count,amountMicro,batch,note}` |
| GET/PUT | `/api/admin/settings` | 站点设置读写 |
| GET | `/api/admin/audit?page=` | 审计日志 |
| GET | `/api/admin/stats/daily?days=30` | 每日收入/成本/请求/新增用户 |
| GET | `/api/admin/export/usage.csv?from=&to=` | CSV 导出（`text(res,200,csv,'text/csv; charset=utf-8')`） |

#### 4.4.1 渠道与模型字段名（实测口径，前后端与文档统一用这套）

```js
// POST/PATCH /api/admin/channels
{
  name: 'openai-main',            // 必填
  provider: 'openai',             // openai | anthropic | gemini
  baseUrl: 'https://api.openai.com/v1',
  apiKey: 'sk-...',               // 仅传入，永不回显；列表返回 apiKeyMasked
  models: ['gpt-4o', 'gpt-4o-mini'],
  modelMap: { 'gpt-4o': 'gpt-4o-2024-08-06' },  // 客户端名 → 上游名
  priority: 10, weight: 100, maxRpm: 0, timeoutMs: 0,
  headers: {}, status: 'active'
}

// POST /api/admin/models
{
  model: 'gpt-4o-mini', displayName: 'GPT-4o mini', vendor: 'openai',
  inputPriceMicro: 1800, outputPriceMicro: 7200,      // 对客价（微元/1K token）
  upstreamInMicro: 1080, upstreamOutMicro: 4320,      // 上游成本
  perRequestMicro: 0, enabled: true, note: ''
}
```

### 4.5 支付（B · `payment.js`）
| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/pay/create` | 同 `/api/recharge` 的底层实现 |
| GET | `/api/pay/notify` | 易支付异步回调：验签 → 幂等置为 paid → 加余额 → 写 ledger → 返回 `success` |
| GET | `/api/pay/return` | 302 到 `/console#/billing` |
| GET | `/api/pay/status/:orderNo` | `{status}` |

`payment.js` 必须导出 `createOrder({userId, amountMicro, payType, ip})` 供 `user.js` 调用；
未配置 `EPAY_PID/EPAY_KEY` 时返回 `{orderNo, payUrl:null, manual:true, message:'仅支持卡密充值'}`，不得抛错。

### 4.6 网关（LEAD · `v1.js`，已完成）
`GET /v1/models`、`POST /v1/chat/completions`、`POST /v1/completions`、`POST /v1/embeddings`。

## 5. 上游适配器契约（A）

`src/upstream/anthropic.js` / `gemini.js` 默认导出：

```js
export default {
  id: 'anthropic',            // 必须与 channels.provider 一致
  label: 'Anthropic',
  passthrough: false,         // 需要协议翻译时为 false
  supportsStreamOptions: false, // 可选：false 时网关不注入 stream_options
  adaptBody(clientBody, upstreamModel, opts) -> upstreamBody,
  buildUrl(channel, {stream, path, model}) -> string,   // model 为上游模型名
  buildHeaders(apiKey, channel) -> Record<string,string>,
  parseResponse(json, {model}) -> {text, finishReason, toolCalls?, usage:{promptTokens, completionTokens}},
  parseStreamEvent(eventName, dataObject) -> NormalizedEvent[],
}
// 同时可命名导出 toOpenAiChatCompletion(raw, {model})：网关用它把非 OpenAI 上游的
// 非流式响应整形成 OpenAI 形状（缺失则原样返回，仅对 passthrough 上游适用）。
```
`NormalizedEvent`：
```js
{type:'delta', text} | {type:'tool_delta', index, id?, name?, argumentsDelta?}
| {type:'usage', promptTokens, completionTokens} | {type:'finish', finishReason} | {type:'ignore'}
```
`src/upstream/index.js` 会自动 `import('./anthropic.js' | './gemini.js')`，存在即注册，
并把命名导出 `toOpenAiChatCompletion` 挂到适配器对象上，无需改注册表。

## 6. 错误码约定

| HTTP | code | 场景 |
|---|---|---|
| 400 | `invalid_request` | 参数缺失/类型错 |
| 401 | `invalid_api_key` | Key 无效/停用/过期 |
| 402 | `insufficient_quota` | 余额或 Key 额度不足 |
| 403 | `forbidden` / `model_disabled` | 权限不足 / 模型下线 |
| 404 | `not_found` / `model_not_found` | — |
| 429 | `rate_limit_exceeded` / `concurrency_limit_exceeded` | 限流 |
| 503 | `no_available_channel` | 无可用渠道 |
| 504 | `upstream_timeout` | 上游超时 |

## 7. 前端约定（C）

- 纯静态，无构建：`public/index.html`（落地/价格）、`public/console.html`（用户台）、`public/admin.html`（后台）。
- 统一 `public/assets/app.css` + 每页一个 `public/assets/<page>.js`。
- 登录态用 Cookie（`rs_sid`，HttpOnly），前端只调 `/api/*`，`fetch(..., {credentials:'include'})`。
- 所有金额显示：`(micro/1e6).toFixed(4) + ' 元'`，微元小于 1 分时保留 4 位。
- 后台入口 `/admin`，用户台 `/console`，`server.js` 已做别名映射。
- 不得在页面里硬编码任何 Key、成本价或上游地址。
