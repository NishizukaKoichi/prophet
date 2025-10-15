# The Prophet — API (Cloudflare Workers)

This is a Cloudflare Workers API (Hono + Durable Object) skeleton that implements the core contracts and business rules from the spec.

- Durable Object: serializes `grant/buy/spend` and tracks an in-memory profile/wallet/ledger per instance.
- Endpoints: map 1:1 to the minimal API contract; Stripe hooks are stubbed but shaped.
- Domain math (XP, level, caps, spend order) lives in `packages/domain` and is consumed from compiled output.

## Quickstart

- Dev: `npm i` then `npm run dev` (requires `wrangler` auth).
- Build: `npm run build`
- Deploy: `npm run deploy`

Headers for local dev (temporary auth stub):
- `x-user-id`: user id (default: `demo-user-1`)
- `x-email`: email (default: `demo@example.com`)

## Bindings (wrangler.toml)

- Durable Object: `LEDGER_DO` → `LedgerDO`
- Vars:
  - `MP_GRANT_MONTHLY=999`
  - `ETHER_GRANT=333`
  - `ETHER_PRICE_CENTS=300`
  - `LEVEL_CAP=100`
  - `JWT_TTL_MIN=5`
  - `ADMIN_SECRET`（管理API用）
  - `MAIL_FROM`, `MAIL_FROM_NAME`（Magic Link送信に使用）
  - `APP_SSO_PRIVATE_KEY_PEM`（Apps SSOのRS256署名; PKCS#8）
  - `STRIPE_SECRET_KEY`, `STRIPE_PRICE_ETHER`, `STRIPE_PRICE_SUB`, `STRIPE_MP_WEBHOOK_SECRET`, `STRIPE_SUBSCRIPTION_WEBHOOK_SECRET`, `PUBLIC_BASE_URL`
  - `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_REGION=auto`, `R2_BUCKET`（R2 署名URLを生成する場合）

## Endpoints

- POST `/api/auth/magic-link` → stub
- GET `/api/auth/callback` → stub
- POST `/api/stripe/subscription-webhook` → `invoice.paid` → grant free(+999); `customer.subscription.deleted` → membership = NONE
- GET `/api/mp/wallet` → `{ free, paid, total, cap, level, currentXp, membership }`
- GET `/api/mp/ledger` → `[{...}]`
- GET `/api/mp/store` (ACTIVE only) → `{ pack:'ether', grant:333, price_cents:300 }`
- POST `/api/mp/buy` → returns mock checkout url; real impl should create Stripe Checkout Session with quantity
- POST `/api/stripe/mp-webhook` → grant paid(+333×quantity) with `payment_intent` idempotency
- POST `/api/articles/:slug/unlock` body `{ cost }` → spend in free→paid order; adds XP (1×)
- POST `/api/market/:id/purchase` body `{ cost }` → spend; adds XP with market multiplier (1.5, integer rounding)
- POST `/api/apps/:id/sso-token` → RS256 署名トークン（`APP_SSO_PRIVATE_KEY_PEM` 設定時。未設定時はstub）
- GET `/api/market/:id/presign?token=...` → ダウンロードトークンから短期限のR2署名URLを発行（発行と同時にトークンは使用済み）

## Notes

- DO is in-memory for now; replace with D1/DB for persistence. Idempotency keys are tracked in-memory.
- Wallet cap is enforced on grant; normal flow (spend→inject) prevents overflow.
- Leveling stops at 100; XP gauge resets to 0 at max.
- UI policy (CTA single) is supported by returning only the info needed to decide between unlock vs top-up/rejoin.

- ダウンロードURL:
  - 厳密な単回性が必要な場合は `download_url`（Worker 経由; DBトークン検証）を使用
  - R2 署名URLはS3仕様上「短期限」にはできるが「単回」自体は保証不可のため、用途に応じて使い分ける

## Secrets 設定例

```
wrangler d1 create prophet_db
wrangler r2 bucket create prophet-assets
wrangler secret put STRIPE_SECRET_KEY
wrangler secret put STRIPE_MP_WEBHOOK_SECRET
wrangler secret put STRIPE_SUBSCRIPTION_WEBHOOK_SECRET
wrangler secret put APP_SSO_PRIVATE_KEY_PEM < private-key.pem
wrangler secret put R2_ACCESS_KEY_ID
wrangler secret put R2_SECRET_ACCESS_KEY
# R2_ACCOUNT_ID はアカウントIDを環境変数または wrangler.toml vars に設定
```
