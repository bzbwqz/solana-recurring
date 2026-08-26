# Solana Recurring

基于 Solana 的自托管订阅、稳定币定期扣款、一次性固定服务额度与 LiteLLM 配额发放 Docker 镜像。

项目建立在 Solana Subscriptions Program 之上。LiteLLM 是第一个支持的服务开通适配器，同时支持 webhook 与 noop 模式。

> 源码维护在私有仓库，不包含在本公开仓库中。本仓库仅提供 Docker 部署与配置说明。

## 功能

- Resend 魔法链接的邀请制邮箱登录。
- 支持 Phantom、Solflare、Backpack 等 Wallet Standard 钱包。
- 定期订阅：用户钱包授权后，商户 puller 每期发起稳定币扣款。
- 一次性固定额度：用户签名将 SPL Token 直接转入商户收款账户。
- LiteLLM 集成：周期扣款成功后写入月度额度；固定额度产品使用长窗口，不会实际按月重置。
- 每位用户只能有一个当前订阅，支持取消与切换套餐。
- PostgreSQL 保存订阅、扣款、购买与 Provisioning 审计记录。

## Docker 镜像

```bash
docker pull ghcr.io/bzbwqz/solana-recurring:v0.1.0
```

生产环境请使用固定版本号，不要只依赖 `latest`。

## Devnet 快速开始

首次使用必须在 Devnet 测试；不要使用主网钱包或真实资金。

1. 创建仅部署用户可读的环境文件：

```bash
sudo install -d -m 700 /opt/solana-recurring
sudo install -m 600 /dev/null /opt/solana-recurring/portal.env
sudoedit /opt/solana-recurring/portal.env
```

2. 填写最小 Devnet 配置：

```dotenv
NODE_ENV=production
PORT=8080
PUBLIC_BASE_URL=http://localhost:8080

DATABASE_URL=postgres://portal:CHANGE_ME@postgres-host:5432/portal
SESSION_SECRET=生成至少32字符的随机值
ADMIN_EMAILS=admin@example.com
ADMIN_WALLETS=

SOLANA_CLUSTER=devnet
SOLANA_RPC_URL=https://api.devnet.solana.com
SUBSCRIPTIONS_PROGRAM_ID=De1egAFMkMWZSN5rYXRj9CAdheBamobVNubTsi9avR44
TOKEN_MINT=你自己的Devnet测试SPL_Token_Mint
TOKEN_DECIMALS=6
TOKEN_SYMBOL=tUSD
TOKEN_PROGRAM_ID=TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
MERCHANT_SECRET_KEY=base58编码的64字节Devnet商户私钥
MERCHANT_RECEIVE_ADDRESS=

BILLING_ENABLED=true
BILLING_CRON=*/10 * * * *

PROVISIONER=noop
RESEND_API_KEY=
EMAIL_FROM=Billing <onboarding@resend.dev>
```

3. 启动容器：

```bash
docker run -d --name solana-recurring --restart unless-stopped \
  -p 8080:8080 \
  --env-file /opt/solana-recurring/portal.env \
  ghcr.io/bzbwqz/solana-recurring:v0.1.0
```

4. 打开 `http://localhost:8080`。若未设置 `RESEND_API_KEY`，登录链接会出现在容器日志：

```bash
docker logs solana-recurring
```

完整 Devnet 测试需要三个独立对象：有少量 SOL 的商户测试钱包、有 Devnet SOL 的订阅用户钱包，以及你自己控制的测试 SPL Token mint。用户钱包必须持有足够测试代币。

## LiteLLM 配置

只有需要 LiteLLM 自动发放额度时才设置：

```dotenv
PROVISIONER=litellm
LITELLM_BASE_URL=https://your-litellm.example.com
LITELLM_ADMIN_KEY=专用LiteLLM管理员Virtual_Key
LITELLM_DEFAULT_BUDGET=1
LITELLM_BUDGET_DURATION=30d
LITELLM_FIXED_BUDGET_DURATION=36500d
LITELLM_SEND_INVITE_EMAIL=false
```

周期订阅成功会写入 Plan 对应额度、清零 `spend` 并使用 `LITELLM_BUDGET_DURATION`。一次性固定额度会写入固定 `max_budget`，并使用 `LITELLM_FIXED_BUDGET_DURATION`，不会实际按月重置。

用户取消周期订阅后，后续链上扣款停止；当前 LiteLLM `max_budget` 与 `spend` 保留，预算窗口改为长固定窗口。

### 从 LiteLLM Model Hub 链接 Portal

部署 Portal 后，可以在 LiteLLM Model Hub 中增加 Portal 付款页面链接，让用户能直接找到订阅与购买固定额度的入口。

1. 使用 LiteLLM 管理员账号登录 proxy。
2. 进入 **AI Hub** -> **Model Hub Table**。
3. 在 **Useful Links Management** 中新增链接，例如：

```text
名称：购买额度 / Billing
地址：https://portal.你的域名/account
```

该链接会显示在公开 Model Hub 页面，不会显示在 Swagger 文档首页。Portal 负责支付和钱包授权，LiteLLM 继续负责服务与 API key。

示例 Model Hub：[LiteLLM Model Hub](https://litellm03.dbapibuilder.com/ui/model_hub_table)

## 生产部署检查

- 使用独立且低余额的商户热钱包，绝不使用个人主钱包。
- 先完成 Devnet 测试，再使用可靠主网 RPC 与主网 USDC mint。
- 使用独立的生产 PostgreSQL，开启 TLS 并建立备份。
- `PUBLIC_BASE_URL` 必须填写公开 HTTPS 地址。
- 配置通过平台 Secret 或服务器私有 `--env-file` 注入；不得写进镜像或 Git 仓库。
- 设置 `ADMIN_WALLETS`，使管理员邮箱登录后还需钱包签名。
- 切换网络或 token mint 后必须重新发布全部 Plan；Devnet 与 Mainnet 链上账户不能迁移。
- 容器必须持续运行，计费 worker 才能每期发起扣款。

## 支持

这是自托管软件。部署者自行负责服务器、数据库、商户钱包、邮件服务与 LiteLLM 实例。

## 赞助

如果项目对你有帮助，可以通过 SOL 或 Solana Token 支持开发：

```text
5u3ZZatCjJ87j28PTPh1ibBvDJXVeTb6DeV88dh3CTGN
```

## 安全

- 不要在 issue、聊天或日志中发送私钥、API key、数据库连接串或 `.env` 文件。
- 商户私钥能够创建 Plan 并发起已授权扣款，必须按生产级秘密管理。
- 使用真实资金前，务必先在 Devnet 完整测试并核对所有钱包签名提示。
