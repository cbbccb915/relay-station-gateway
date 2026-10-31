<!-- PUBLIC_URL_MARKER: https://donor-addressing-sea-conditioning.trycloudflare.com -->
> 在线交换台：<https://donor-addressing-sea-conditioning.trycloudflare.com/access>　（稳定域名，GitHub Issue 提交也会自动进复核队列）

# Relay Station · OpenAI 兼容中转站

一个**零依赖**（Node 24 + `node:http` + `node:sqlite`）的 OpenAI 兼容 API 网关，
带完整计费、渠道故障转移、号池轮换、双通道充值与对账。

> 生产可用的小型商业化底座：单进程、单文件 SQLite、无 npm 依赖、无构建步骤。

## 能做什么

| 能力 | 说明 |
|---|---|
| OpenAI 兼容入口 | `POST /v1/chat/completions`、`/v1/embeddings`、`/v1/models`，流式与非流式 |
| 协议翻译 | 上游是 anthropic / gemini 风格时，自动把请求/响应/SSE 事件转成 OpenAI 形状 |
| 预扣-结算 | 请求前按 `prompt + max_tokens×输出价` 预扣，结束后按真实 usage 多退少补 |
| 双边账 | `usage_logs` + `ledger` + `reservations` 三张表，任何异常都能对上账 |
| 渠道故障转移 | 按优先级/权重选渠道，429/401/403 时先换 Key 再换渠道，失败自动退款 |
| 号池 | 每渠道多 Key、加密存储、按健康度与冷却时间轮换 |
| 计费 | 微元记账（1 元 = 1,000,000 微元），价目表可在线改，支持输入/输出分别定价 |
| 充值 | 卡密 + 在线支付（易支付）+ 收款码半自动（唯一应付金额 + 尾数对账） |
| 账户 | 邮箱验证码注册、邮箱找回密码、同浏览器多账号切换（上限 3） |
| 运维 | 对账脚本、备份、健康检查、指标、维护任务（分批清理不阻塞事件循环） |

## 为什么不用现成的

- **零依赖**：`npm install` 不装任何包，审计面小，部署就是把目录拷过去。
- **钱的事都写成事务**：预扣、结算、退款、入账、调账全在 `BEGIN IMMEDIATE` 事务里，
  失败时余额和流水不会各说各话。
- **回归测试就是账本**：121 条断言，覆盖双退款、并发扣费、回调幂等、改密撤会话、
  订单过期、令牌限次等，跑一遍等于对一次账。

## 架构

```
src/
  server.js             HTTP 入口、静态站点、健康检查、优雅退出
  http/router.js        极简路由（通配 + 二次解码兜底）
  middleware/auth.js    会话 / API Key / 管理员
  billing/quota.js      预扣、结算、退款、心跳、遗留回收
  billing/pricing.js    价目表与成本核算
  routes/v1.js          网关主体（渠道×Key 双循环、SSE 解析、用量落库）
  routes/payment.js     易支付 + 收款码 + 订单状态
  routes/access.js      资料交换（目录 / 提交 / 评分 / 令牌下载）
  services/             计费视图、统计、邮件、验证码、收款码、访问交换
  jobs/retention.js     维护任务（清理日志、回收预扣、过期订单）
public/                 控制台、管理后台、落地页、交换台、复核台
scripts/                对账、漂移检测、备份、演练、打包
tests/                  协议、号池、支付、资金并发、验收、回归
```

## 快速开始

```bash
cp .env.example .env       # 填 RELAY_ADMIN_EMAIL / RELAY_ADMIN_PASSWORD
node src/server.js         # 默认 http://127.0.0.1:8080
```

首启会创建 `data/relay.db`、`data/master.key`，并把管理员账号写进库。

## 想拿完整源码 / 配套资料？

本仓库只放**架构与公开文档**（`docs/`）。完整源码、运维脚本包、以及若干配套资料
走**交换制**——用等价情报换，或者付 **0.1 USD** 等值：

见 [ACCESS.md](ACCESS.md)。

## 许可

本仓库文档与架构说明：MIT。完整源码包的许可随包发放（含商业使用条款）。

## 免责声明

本项目用于自建 API 网关与计费系统。接入第三方模型服务时请自行确认其条款；
上游服务的可用性、定价与合规由服务提供方决定。
