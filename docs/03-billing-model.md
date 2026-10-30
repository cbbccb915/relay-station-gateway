# 03 · 计费与定价模型

> 铁律：**所有金额都是整数微元**（`1 元 = 1_000_000 微元`），代码里不出现浮点运算。
> 单位换算只发生在两个地方：后台表单输入 / 前端展示（`micro / 1e6`）。

## 0. 客户最常问的两件事：为什么"余额够却报错"

这两条是**设计如此**，客服话术请照下面解释，不要当成 bug：

### 0.1 预扣按输出上限估算，且**会显式写到请求里**

调用前先冻结一笔钱（`estimateReserveMicro`）：

```
预扣 = 输入 token 估算 × 输入单价 + 输出上限 × 输出单价
```

**输出上限的规则（2026-10-28 调整）**：

| 客户端传了什么 | 网关怎么做 |
|---|---|
| 三个字段都没传 | **自动补 `max_tokens = 4096`**，并按 4096 预扣 |
| 传了 `max_tokens` / `max_completion_tokens` / `max_output_tokens` 其中一个 | 用它，超过 64000 夹到 64000 |
| 传了多个且值不同 | 取**最小值**，并把这些字段都改成同一个数 |

为什么要"自动补"：以前只在小本本上按 1024 算预扣、却**不把上限发给上游**，
上游可以按自己的上限产出，于是出现「预扣 1024、实际产出 3 万 token」的透支
（Codex 审查实测余额被打到 −36000）。现在预扣和上游收到的是同一个数。

> 提示客户：**需要长输出时请显式传 `max_tokens`**（如 `max_tokens: 8000`，上限 64000）。
> 不传就是 4096，超出的部分会被上游截断。

### 0.2 每把 Key 默认并发 8，第 9 并发直接 429

`429 concurrency_limit_exceeded` 不是限速错误，是**同时进行中的请求数**超了。
默认值在"系统设置 → 默认并发数"（`default_concurrent`，初始 8），可以按 Key 单独调
（后台 → API Keys → 编辑 → 并发上限；填 `0` 表示跟随全局默认）。

> 提示客户：并发不够就调高这个值，或者把这批请求串行化；`429` 的那几笔**不扣费**。

### 0.3 DeepSeek 默认开"思考模式"，`max_tokens` 太小会得到空回复

实测（2026-10-28，`deepseek-chat` 映射到 `deepseek-flash`）：

| max_tokens | 结果 |
|---|---|
| 32 | `content="你好世界"`，`reasoning_content` 38 字 |
| 64 | **`content` 为空**，64 个输出 token **全部**被 reasoning 吃掉（`reasoning_tokens: 64`）|

也就是说：输出预算会先被"思考"消耗，剩不下才写正文。客户看到"返回 200 但内容是空"，多数是这个原因。

> 客服话术：把 `max_tokens` 给到 **256 以上**；或在请求里关闭思考模式（DeepSeek 文档的思考模式开关）。
> 计费照实际 token 走，不会多收，但"空回复"会被误当成故障。

## 1. 单位

| 名称 | 值 | 用在哪 |
|---|---|---|
| 微元 micro | 最小货币单位 | 数据库所有金额列、API 所有金额字段 |
| 1 元 | 1 000 000 微元 | 展示层 `(micro/1e6).toFixed(4)` |
| 计价单位 | **每 1K token** | `model_prices.*_micro` 全部是"微元 / 1K token" |

举例：`gpt-4o-mini` 输入价 `1800` 微元/1K = `0.0018 元/1K` = 每 1000 输入 token 收 0.18 分。

## 2. 计费公式

`src/billing/pricing.js`：

```js
// 不足 1K 按比例折算，四舍五入到整微元
priceFor(tokens, pricePer1kMicro) = Math.round(tokens * pricePer1kMicro / 1000)

chargeMicro = priceFor(promptTokens,     input_price_micro)
            + priceFor(completionTokens, output_price_micro)
            + per_request_micro                      // 按次附加费，默认 0

costMicro   = priceFor(promptTokens,     upstream_in_micro)
            + priceFor(completionTokens, upstream_out_micro)
            // 注意：per_request_micro 不计入成本

marginMicro = chargeMicro − costMicro
```

三个必须知道的细节：

1. **四舍五入在最终微元上做**，不是先算元再乘回来 —— 所以 1 token 也可能产生 1 微元以内误差，长期看正负抵消。
2. **`per_request_micro` 只加在收入、不加在成本**：设 1 元/次（`1000000`）时，这笔钱 100% 是毛利。
3. **上游没返回 usage 时**：prompt 用 `estimateRequestTokens` 估算，completion 用 `estimateCompletionTokens`（按字符：中日韩 0.75 token/字，其它 1/3.6 token/字符）。**流式请求如果上游不返回 usage，输出 token 是估算值** —— 这是全站唯一的口径风险点，尽量选支持 `stream_options.include_usage` 的上游。

### 算例（gpt-4o-mini，一次非流式调用）

| 项 | 值 |
|---|---|
| prompt_tokens | 1000 |
| completion_tokens | 500 |
| 输入价 / 成本 | 1800 / 1080 微元/1K |
| 输出价 / 成本 | 7200 / 4320 微元/1K |
| 收入 | `round(1000×1800/1000) + round(500×7200/1000)` = 1800 + 3600 = **5400 微元 = 0.0054 元** |
| 成本 | 1080 + 2160 = **3240 微元 = 0.00324 元** |
| 毛利 | **2160 微元 = 0.00216 元**（40%） |

## 3. 预扣与结算

路由层（`src/routes/v1.js`）的顺序是：**先估算冻结，再打上游，最后按真实用量多退少补**。

```
① estimateReserveMicro(price, body)
     inputTokens = estimateRequestTokens(body)                 // 估算，不是真实值
     maxOut      = body.max_completion_tokens ?? max_tokens ?? max_output_tokens ?? 4096
     maxOut      = min(maxOut, 64000)                          // 单次封顶
     reserveMicro = max(1, calcCharge(price, inputTokens, maxOut).chargeMicro)

② reserveQuota()   →  Key 额度校验 + 原子扣减 users.balance_micro
                      失败：402 insufficient_quota，不发起上游请求

③ 上游调用

④ settleQuota()    →  chargeMicro = calcCharge(真实 usage)
                      refundMicro = reserved − charge          // 可正可负
                      users.balance_micro += refundMicro
                      api_tokens.used_micro += chargeMicro
                      usage_logs 插一行（含 charge/cost/延迟/状态）
                      ledger 插一行 kind='usage', amount = −chargeMicro

⑤ 失败路径 refundQuota() → 全额退回预扣，usage_logs 记一行 charge=0 的失败日志
```

关键行为：

| 场景 | 结果 |
|---|---|
| 预估高于实际 | 差额退回余额（`refundMicro > 0`） |
| 预估低于实际（上游超出 max_tokens、估算偏低） | `refundMicro < 0`，**会继续扣，余额可能变成负数** |
| 请求失败 | 全额退回，不产生 `ledger.usage` 行 |
| 流式中途断线 | 按已产生用量结算（`status='partial'`），不足预扣的部分退回 |

> **负余额**是真实存在的：`settleQuota` 不再校验余额。运营上必须每天扫一次
> `SELECT id,email,balance_micro FROM users WHERE balance_micro < 0;`，
> 对负余额账户停 Key、追缴或销账。控制手段是把 `default_rpm` 调低、把 Key 的 `quota_micro` 设成有限值。

## 4. 倍率与毛利

```
单模型输入倍率 = input_price_micro  / upstream_in_micro
单模型输出倍率 = output_price_micro / upstream_out_micro

输入毛利率 = 1 − upstream_in_micro  / input_price_micro
输出毛利率 = 1 − upstream_out_micro / output_price_micro

整体毛利率 = Σ(charge) − Σ(cost)  /  Σ(charge)
```

默认价目表的实际倍率（详见 [定价毛利测算.csv](定价毛利测算.csv)）：

| 模型档 | 倍率 | 毛利率 |
|---|---|---|
| gpt-4o-mini | 1.67 | 40.0% |
| gpt-4o / gpt-4.1 系 | 1.60 | 37.4–37.5% |
| deepseek-chat / reasoner、claude 系、qwen-max | 1.50 | 33.3% |
| gemini-2.5-flash | 1.57 | 36.5% |

**为什么不是越高越好**：倍率 2.0 时毛利率 50%，但同类中转站普遍在 1.3–1.8 之间出价，倍率过高直接失去价格竞争力。建议：**主力走量模型 1.5–1.6，长尾/稀缺模型 1.8–2.2，embedding 可以做到 3.0 以上（金额太小客户不敏感）**。

## 5. 改价操作步骤

价格是**读时生效**的：`requireModelPrice()` 每次请求都查 `model_prices`，改完立刻对新请求生效，**不需要重启服务**。已经在跑的长连接按旧价结算。

### 方式 A · 后台页面（推荐）

后台 →「模型」页 → 找到模型 → 改「售价（元/1K）」「成本（元/1K）」→ 保存。
表单里填元，前端换算成微元后调 `POST /api/admin/models`。

### 方式 B · 接口

```powershell
# 先把元换算成微元：0.0018 元/1K → 1800 微元/1K
# PowerShell 5.1 直接内联 JSON 会丢引号，这里把 body 写文件再 -d "@文件"
[IO.File]::WriteAllText("$env:TEMP\model.json", '{"model":"gpt-4o-mini","displayName":"GPT-4o mini","vendor":"openai","inputPriceMicro":1800,"outputPriceMicro":7200,"upstreamInMicro":1080,"upstreamOutMicro":4320,"enabled":true}')
curl.exe -s -b cookies.txt -H "Content-Type: application/json" -X POST `
  http://127.0.0.1:8081/api/admin/models -d "@$env:TEMP\model.json"
```

> `cookies.txt` 来自登录：`curl.exe -s -c cookies.txt -X POST .../api/auth/login -d "@login.json"`。
> 更省事的写法是用 `Invoke-WebRequest`（见 [06-上线验收清单](06-上线验收清单.md) 第 0 节的 `Api` 函数）。

### 方式 C · 直接改库（应急，改完无需重启）

```powershell
node --disable-warning=ExperimentalWarning scripts/report.mjs --allow-write --sql 'UPDATE model_prices SET input_price_micro = ?, output_price_micro = ?, updated_at = strftime(''%Y-%m-%dT%H:%M:%fZ'',''now'') WHERE model = ?' --param 2000 --param 8000 --param gpt-4o-mini
```

改完立刻核对：

```powershell
node --disable-warning=ExperimentalWarning scripts/report.mjs --sql 'SELECT model,input_price_micro,output_price_micro FROM model_prices WHERE model = ''gpt-4o-mini'''
```

> 注意 SQL 里的单引号要写成 `''`（SQL 转义）。嫌麻烦就直接用方式 A 的后台页面。

### 改价前后必做

1. **记一次基线**：把当前 `model_prices` 全表导出，改价出问题时能立刻回滚。
   ```powershell
   node --disable-warning=ExperimentalWarning scripts/report.mjs --sql 'SELECT * FROM model_prices' --json |
     Set-Content "backups\model_prices-$(Get-Date -Format yyyyMMdd-HHmmss).json" -Encoding utf8
   ```
2. **通知已付费客户**：调价只影响新请求，但客户看到账单变化会来问。
3. **盯 24 小时毛利率**：改价后如果毛利率掉到 20% 以下，八成是上游采购价变了或者渠道被换成了贵的。

## 6. 对账口径

每天（或每周）跑一遍，四个数能对上才算账干净：

```powershell
node --disable-warning=ExperimentalWarning scripts/report.mjs --days 30
```

输出里直接有：调用数/失败率、token、收入、成本、毛利、毛利率、余额负债、累计充值、
**账本一致性校验**（ledger 消费流水 vs usage_logs 扣费）、负余额账户、同 IP 多账号。
接监控用 `--json`。

单条校验（两条应该相等）：

```powershell
node --disable-warning=ExperimentalWarning scripts/report.mjs --sql 'SELECT (SELECT -SUM(amount_micro) FROM ledger WHERE kind=''usage'') AS ledger_usage, (SELECT SUM(charge_micro) FROM usage_logs WHERE status IN (''ok'',''partial'')) AS usage_charge'
```
```

| 指标 | 定义 | 健康值 |
|---|---|---|
| 收入 | `Σcharge_micro`（ok + partial） | — |
| 上游成本 | `Σcost_micro` | — |
| 毛利率 | (收入−成本)/收入 | 目标 30–45% |
| 余额负债 | `Σusers.balance_micro` | 应 ≈ 累计充值 − 累计消费（误差 < 1 元说明账本对得上） |
| 失败率 | `status='error'` 占比 | < 3%，持续偏高说明渠道有问题 |
| 估算结算占比 | 上游没返回 usage 的请求 | 越低越好，`>10%` 要查上游是否支持 `include_usage` |
| 负余额数 | `balance_micro < 0` 的用户数 | 0（出现即处理） |

一致性校验 SQL（两条应该相等）：

```sql
-- 账本里的消费流水总额
SELECT -SUM(amount_micro) FROM ledger WHERE kind = 'usage';
-- 用量表里的扣费总额
SELECT SUM(charge_micro)  FROM usage_logs WHERE status IN ('ok','partial');
```

## 7. 异常单处理

### 7.1 半流断线（`status = 'partial'`）

现象：客户端已经收到部分内容，上游/网络断了。系统行为：按 `estimateRequestTokens(body)` 结算输入 token，**输出记 0**，`usage_logs.error` 里留原因，客户端收到一条 `upstream_disconnected`。

处理：
- 客户投诉"没答完却扣了钱"：查 `usage_logs` 该 `request_id`，若 `completion_tokens = 0` 且 `charge_micro` 很小，可直接手工补偿：
  ```sql
  -- 先看清楚
  SELECT request_id,model,prompt_tokens,completion_tokens,charge_micro,status,error
  FROM usage_logs WHERE request_id = '客户提供的 x-request-id';
  ```
- 补偿用后台「用户详情 → 加减款」，写 `ledger(kind='adjust')` 并留 `note`。**不要直接 UPDATE users**，否则账本和余额对不上。

### 7.2 上游 5xx / 429 重试

- 非流式：同一请求最多尝试 `settings.upstream_retry + 1` 次（默认 3 次），换渠道；全部失败则**全额退款**并返回 503。
- 流式：开始输出前失败 → 换渠道；开始输出后失败 → 按 partial 结算，不再换渠道（否则客户端会收到两段内容）。
- 重试次数记在 `usage_logs.retries`，渠道失败会扣健康分（−12）并在 30 秒内不再被选中。
- 运营动作：`SELECT id,name,health_score,fail_count,last_error FROM channels ORDER BY health_score ASC;` 健康分 < 50 的渠道要查 Key 是否过期、余额是否耗尽。

### 7.3 上游没返回 usage

- 表现：`usage_logs.prompt_tokens` 等于估算值，可能和上游账单差 10–20%。
- 处理：优先换支持 `stream_options.include_usage` 的渠道；对流式请求，`src/routes/v1.js` 会自动加上 `stream_options.include_usage=true`，若上游 400 报错说 `stream_options` 不支持，会在同渠道降级重试一次（日志 `stream_options_unsupported`）。

### 7.4 重复扣费（支付回调重放）

- 支付回调必须幂等：`orders.status` 已是 `paid` 就直接返回 `success`，不重复加余额。
- 自查：
  ```sql
  SELECT out_trade_no, COUNT(*) c FROM ledger WHERE kind='recharge' GROUP BY ref HAVING c > 1;
  ```
  出现重复行说明幂等逻辑被绕过，手工冲正（`adjust` 负数流水）+ 修代码。

### 7.5 客户坚持"扣多了"

标准动作（三步）：

1. 要 `x-request-id`（或让客户给请求时间点），查 `usage_logs`。
2. 用 `calcCharge` 复算一遍，把 `model / prompt_tokens / completion_tokens / 单价` 摆给客户看。
3. 差价 > 0.01 元且确属我方原因（估算偏差、上游 usage 缺失）→ 补偿；否则解释清楚并给对账明细。

补偿命令（后台「用户详情 → 加减款」等价）：

```powershell
# 50000 微元 = 0.05 元
[IO.File]::WriteAllText("$env:TEMP\adjust.json", '{"deltaMicro":50000,"note":"补偿：request 8f3c... 半流断线"}')
curl.exe -s -b cookies.txt -H "Content-Type: application/json" -X POST `
  http://127.0.0.1:8081/api/admin/users/2/balance -d "@$env:TEMP\adjust.json"
```

## 8. 定价红线（给老板的数字）

| 指标 | 底线 | 说明 |
|---|---|---|
| 综合毛利率 | ≥ 25% | 低于这个数，一次上游涨价或汇率波动就亏 |
| 单模型最低毛利率 | ≥ 15% | 引流款可以低，但不能为负 |
| 免费额度成本 | ≤ 拉新预算的 30% | 注册赠送（`RELAY_REGISTER_GIFT_MICRO`）按 gpt-4o-mini 算，1 元 ≈ 55 万输入 token |
| 支付手续费 | 2%（易支付常见档） | 记在毛利之后，别忘算 |

盈亏平衡公式：

```
盈亏平衡日 token 量 = (月固定成本 + 月支付手续费) / (每 token 毛利 × 30)
```

按默认价目 + 2C4G 机器（60 元/月）+ 域名（5 元/月）+ 2% 手续费、混合用量测算：
**约 49 万 token/日** 打平。详细场景见 [04-运营手册](04-运营手册-获客与毛利.md#5-毛利测算表真实数字)。
