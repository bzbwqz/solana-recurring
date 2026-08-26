# Solana Recurring

[English](README.md) | [简体中文](README.zh-CN.md)

Self-hosted Docker distribution for Solana subscriptions, recurring stablecoin payments, one-time fixed service credits, and LiteLLM entitlement updates.

Built on the Solana Subscriptions Program. LiteLLM is the first supported entitlement provider; webhook and noop provisioners are also available.

> Source code is maintained in a private repository and is not included here. This public repository contains deployment documentation only.

## What It Does

- Invite-only email login with Resend magic links.
- Wallet Standard support for Phantom, Solflare, Backpack, and compatible wallets.
- Recurring subscription plans: user wallet authorizes a merchant puller to collect a stablecoin amount each period.
- One-time fixed credit products: user signs a direct SPL Token payment to the merchant receiving account.
- LiteLLM integration: successful recurring charges issue a monthly budget; fixed credit products use a long non-recurring-equivalent budget window.
- One current subscription per user, with user cancellation and plan switching.
- PostgreSQL billing, purchase, and provisioning audit records.

## Docker Image

```bash
docker pull ghcr.io/bzbwqz/solana-recurring:v0.1.0
```

Use a version tag in production. Do not rely solely on `latest`.

## Quick Start: Devnet

Do not use a mainnet wallet or real funds for first-time testing.

1. Create a private runtime directory and environment file:

```bash
sudo install -d -m 700 /opt/solana-recurring
sudo install -m 600 /dev/null /opt/solana-recurring/portal.env
sudoedit /opt/solana-recurring/portal.env
```

2. Configure the minimum Devnet variables:

```dotenv
NODE_ENV=production
PORT=8080
PUBLIC_BASE_URL=http://localhost:8080

DATABASE_URL=postgres://portal:CHANGE_ME@postgres-host:5432/portal
SESSION_SECRET=GENERATE_AT_LEAST_32_CHARACTERS
ADMIN_EMAILS=admin@example.com
ADMIN_WALLETS=

SOLANA_CLUSTER=devnet
SOLANA_RPC_URL=https://api.devnet.solana.com
SUBSCRIPTIONS_PROGRAM_ID=De1egAFMkMWZSN5rYXRj9CAdheBamobVNubTsi9avR44
TOKEN_MINT=YOUR_DEVNET_TEST_SPL_TOKEN_MINT
TOKEN_DECIMALS=6
TOKEN_SYMBOL=tUSD
TOKEN_PROGRAM_ID=TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
MERCHANT_SECRET_KEY=BASE58_ENCODED_64_BYTE_DEVNET_SECRET_KEY
MERCHANT_RECEIVE_ADDRESS=

BILLING_ENABLED=true
BILLING_CRON=*/10 * * * *

PROVISIONER=noop
RESEND_API_KEY=
EMAIL_FROM=Billing <onboarding@resend.dev>
```

3. Start the container:

```bash
docker run -d --name solana-recurring --restart unless-stopped \
  -p 8080:8080 \
  --env-file /opt/solana-recurring/portal.env \
  ghcr.io/bzbwqz/solana-recurring:v0.1.0
```

4. Open `http://localhost:8080`. Without `RESEND_API_KEY`, login links are emitted in container logs:

```bash
docker logs solana-recurring
```

For a complete Devnet test, use a separate merchant wallet with a small SOL balance, a separate subscriber wallet with Devnet SOL, and a test SPL Token mint that you control. The subscriber needs sufficient test tokens for the chosen plan.

## LiteLLM Configuration

Set these values only when LiteLLM provisioning is desired:

```dotenv
PROVISIONER=litellm
LITELLM_BASE_URL=https://your-litellm.example.com
LITELLM_ADMIN_KEY=YOUR_DEDICATED_ADMIN_VIRTUAL_KEY
LITELLM_DEFAULT_BUDGET=1
LITELLM_BUDGET_DURATION=30d
LITELLM_FIXED_BUDGET_DURATION=36500d
LITELLM_SEND_INVITE_EMAIL=false
```

Recurring payments write the plan budget, reset spend, and use `LITELLM_BUDGET_DURATION`. One-time fixed credit products write the fixed budget and use `LITELLM_FIXED_BUDGET_DURATION`, so they do not practically reset every month.

When a user cancels a recurring subscription, future billing stops. The current LiteLLM budget and spend are preserved, while the budget window is moved to the configured long fixed-credit window.

### Link the Portal from LiteLLM Model Hub

After deploying the Portal, add its public billing page as a LiteLLM Model Hub
useful link so users can discover where to buy subscriptions or fixed credits.

1. Sign in to your LiteLLM proxy as an administrator.
2. Open **AI Hub** -> **Model Hub Table**.
3. In **Useful Links Management**, add a link such as:

```text
Label: Buy credits / Billing
URL: https://portal.your-domain.example/account
```

Users will see the link on the public Model Hub page. It does not appear on the
Swagger documentation home page. The Portal remains the payment and wallet
authorization application; LiteLLM remains the service and API-key provider.

Example Model Hub: [LiteLLM Model Hub](https://litellm03.dbapibuilder.com/ui/model_hub_table)

## Production Checklist

- Use a dedicated low-balance merchant hot wallet, never a personal wallet.
- Use a reliable mainnet RPC and mainnet USDC mint only after Devnet testing.
- Use a separate production PostgreSQL database with backups and TLS.
- Set `PUBLIC_BASE_URL` to your public HTTPS URL.
- Store configuration in platform secrets or a private `--env-file`; never in an image or Git repository.
- Set `ADMIN_WALLETS` to require wallet proof in addition to email access.
- Re-publish all plans when changing cluster or token mint: chain accounts do not migrate between Devnet and Mainnet.
- Keep the container running continuously; the billing worker initiates each recurring collection.

## Support

This is self-hosted software. You operate your own infrastructure, database, merchant wallet, email provider, and LiteLLM instance.

## Sponsor

If this project is useful, you may support its development with SOL or Solana tokens:

```text
5u3ZZatCjJ87j28PTPh1ibBvDJXVeTb6DeV88dh3CTGN
```

## Security

- Never send private keys, API keys, database URLs, or `.env` files through issues or chat.
- A merchant secret key can create plans and initiate authorized collections. Treat it as production secret material.
- Review all wallet signature prompts and test on Devnet before using real funds.
