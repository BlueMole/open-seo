# BlueMole self-host deploy notes (BLU-118)

Fork of [`every-app/open-seo`](https://github.com/every-app/open-seo) (MIT, `homepage: openseo.so`,
matching the `@openseo_so` project named in BLU-118). Upstream is wired as the `upstream` remote.

Purpose: give BlueMole agents real SEO data (keyword research, backlinks, audits) over MCP,
instead of paying for Ahrefs/Semrush. Directly feeds the whodisapp.com SEO backlog
(BLU-39 sitemap hygiene, BLU-41 thin content on money pages).

**Nothing here touches whodisapp.com.** This is a standalone tool deployment.

## Status: blocked, pre-deploy

Verified on 2026-10-21 against upstream `c5f8db1`.

Both blocking gates below were reproduced with the repo's own preflight
(`node scripts/selfhost-deploy-preflight.mjs`), not assumed.

## Deploy path (docs/SELF_HOSTING_CLOUDFLARE.md)

```bash
corepack enable && pnpm install
cp deploy/.env.selfhost.example .env.selfhost   # already done; values still blank
pnpm alchemy cloudflare bootstrap               # deploys alchemy's state-store Worker
pnpm deploy:selfhost --yes
```

`deploy:selfhost` provisions D1 + KV + R2 + two Workers (`open-seo-selfhost`,
`open-seo-selfhost-audit`) + Workflows + Durable Objects, applies migrations, and creates the
Cloudflare Access application that gates login. Teardown is
`pnpm alchemy destroy --env-file .env.selfhost --stage selfhost`.

MCP endpoint after deploy: `https://<worker-hostname>/mcp`. It additionally requires turning on
**Managed OAuth** on the Access application (Zero Trust -> Access controls -> Applications -> Edit ->
Additional settings -> OAuth) and allowlisting loopback redirect URIs, or MCP clients log in but
expose zero tools. See `docs/SELF_HOSTING_CLOUDFLARE_OPERATIONS.md`.

## Blocker 1 — DataForSEO credentials absent (owner: Kevin)

BLU-118 step 4 says to stop if this is missing. It is missing: the Paperclip company secret
catalog has no DataForSEO entry. Reproduced:

```
deploy:selfhost preflight failed:
  DATAFORSEO_API_KEY is not set in .env.selfhost
```

Unblock: Kevin creates the DataForSEO account, copies the **Base64** credential from
<https://app.dataforseo.com/api-access> ("Send by email" -> the longer Base64 value, which is
`email:password` base64-encoded), and stores it in Paperclip Secrets scoped to Webby.
Suggested key: `dataforseo_api_key`. New accounts carry $1 free credit; minimum top-up is $50.
Per BLU-118 the agent must not create the account or enter payment details.

## Blocker 2 — no Cloudflare deploy credential (owner: Locky, then Kevin for R2)

Every Paperclip secret fetch this run returned
`403 Secret access is not granted for this agent` — including `cloudflare_api_token`,
`cloudflare_authorization_bearer` and `account_id`. So there is currently no Cloudflare access at
all, and the previously provisioned token was read-only anyway, which cannot deploy. Reproduced:

```
deploy:selfhost preflight failed:
  No Cloudflare login found (alchemy profile "default")
```

**This does not need Kevin at a browser.** `scripts/selfhost-deploy-preflight.mjs:63` —
"An explicit API token bypasses login profiles entirely" — so setting `CLOUDFLARE_API_TOKEN`
in the environment replaces the interactive `pnpm alchemy login` OAuth flow.

Account-scoped token permissions the selfhost deploy needs:

| Permission | Why |
|---|---|
| Workers Scripts: Edit | both Workers, Durable Objects, Workflows, alchemy state-store worker |
| Workers KV Storage: Edit | KV namespaces |
| D1: Edit | database + migrations |
| Workers R2 Storage: Edit | R2 bucket |
| Account Settings: Read | account/subdomain resolution |
| Access: Apps and Policies: Edit | the Access login gate |

The last row can be dropped by setting `TEAM_DOMAIN` + `POLICY_AUD` in `.env.selfhost` against a
hand-managed Access application — the preflight then provisions no Access resources and does not
need the `access:write` scope. Scoping the token to one account and no DNS/billing keeps it inside
Webby's BLU-35 guardrails.

## Cost correction — BLU-118's "$0 hosting" needs three caveats

1. **R2 activation requires a payment method on file** on the Cloudflare account, even inside the
   free tier (`docs/SELF_HOSTING_CLOUDFLARE.md` prerequisites). Billing is out of Webby's scope —
   Kevin must enable R2 once in the dashboard.
2. Core hosting does run on the Cloudflare **free** plan. But the site-audit
   **Render JavaScript** option needs a **paid Workers plan** ($5/mo; includes 10 browser
   hours, then $0.09/browser-hour). Without it, audits of JS-rendered pages are degraded.
3. DataForSEO is pay-per-call and billed directly by the provider — $50 minimum top-up after the
   $1 trial credit. The $0 refers to Cloudflare hosting only, not to running the tool.

## Resume checklist

1. Kevin: DataForSEO account -> Base64 credential -> Paperclip Secret scoped to Webby.
2. Kevin: enable R2 on the Cloudflare account (one-time, needs a card on file).
3. Locky: provision a write-scoped `CLOUDFLARE_API_TOKEN` per the table above, scoped to Webby.
4. Webby: fill `.env.selfhost`, run `pnpm deploy:selfhost --yes`, record the Worker URL.
5. Webby: enable Managed OAuth on the Access app, verify `https://<worker>/mcp` answers an agent query.
6. Webby: run exactly ONE keyword query for a whodisapp.com money-page keyword and report its exact
   DataForSEO cost on BLU-118. No bulk runs without Kevin's explicit approval.
