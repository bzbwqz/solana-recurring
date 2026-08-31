---
name: portal-deployment-operator
description: "Use when: deploying, upgrading, configuring, validating, or troubleshooting the Solana Recurring Portal on Devnet, Mainnet, Docker, Azure App Service, or Azure Container Apps. Guides secret-safe preflight and requires explicit confirmation before production changes."
argument-hint: "Describe the target: devnet Docker, production Docker, Azure App Service, or Azure Container Apps."
---

# Portal Deployment Operator

This public Skill helps an Agent guide deployment of the Solana Recurring Portal.
It does not give an Agent access to secrets, private keys, merchant wallets, or
permission to deploy production changes without the wallet owner's confirmation.

## Safe Workflow

1. Start with Devnet and a separate test database, merchant wallet, test mint,
   subscriber wallet, and webhook secret.
2. Keep production values in a private environment file or cloud secret store.
   Never paste them into chat, source control, Docker images, or IaC files.
3. Run the Portal's secret-safe Doctor:

```bash
pnpm deployment:doctor -- --config-file=/opt/solana-recurring/portal.env
```

4. Before production mutation, show a non-secret summary: image tag/digest,
   public URL, cluster, mint, merchant public address, database host, billing
   state, enabled adapters, and webhook hosts. Obtain explicit approval naming
   the target environment.
5. Verify `https://<portal-host>/healthz` after deployment and record only
   non-sensitive evidence.

## Platform Rules

- Docker VM: use a fixed image tag or digest and an env file outside the source
  checkout with mode `600`.
- Azure App Service: use application settings or Key Vault references, set
  `WEBSITES_PORT=8080`, and enable Always On.
- Azure Container Apps: use Key Vault plus managed identity; set target port
  `8080`, `minReplicas: 1`, and `maxReplicas: 1` until distributed worker
  coordination is implemented and tested.
- The Portal worker is the only recurring scheduler. No Odoo cron, Agent, or
  external SaaS may call `transferSubscription`.

## Never Do

- Never request or print `.env`, private keys, seed phrases, database URLs, API
  keys, or webhook secrets.
- Never deploy Mainnet or turn on billing without an explicit confirmation that
  names the target environment.
- Never treat a browser redirect or received webhook as settlement confirmation;
  the Portal server must verify settlement on chain.