# WhatsApp Automation for Shopify Stores

Phase A implementation for the MVP SaaS:

- merchant authentication
- merchant onboarding
- Shopify store provisioning
- Shopify OAuth connection
- encrypted access-token storage
- default webhook subscription registration
- Shopify new-order webhook ingestion
- WhatsApp order confirmation message sending
- COD verification automation with inbound YES/NO handling

## Stack

- Next.js App Router
- TypeScript
- Tailwind CSS
- Prisma ORM
- PostgreSQL

## Project Structure

```text
src/
  app/
    api/
      auth/                         # Signup, login, logout
      onboarding/                   # Merchant workspace provisioning
      integrations/shopify/         # OAuth install/callback/status and webhook placeholders
    dashboard/                      # Merchant dashboard
    login/                          # Login page
    onboarding/                     # Merchant onboarding page
    signup/                         # Signup page
  components/
    dashboard/                      # Dashboard cards
    forms/                          # Auth + onboarding forms
    ui/                             # Shared UI primitives
  lib/
    auth/                           # Session cookie handling
    crypto/                         # AES-GCM encryption and HMAC helpers
    db/                             # Prisma singleton
    env/                            # Server env validation
    http/                           # JSON response helpers
    utils/                          # Shared utility helpers
  modules/
    accounts/                       # Merchant account and onboarding services
    shopify/                        # Shopify OAuth and webhook registration services
prisma/
  schema.prisma                     # Multi-tenant schema for Phase A
```

## Environment Variables

Copy `.env.example` to `.env.local` and fill in:

```bash
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/wa_shopify_mvp?schema=public"

NEXT_PUBLIC_APP_URL="http://localhost:3000"
SESSION_SECRET="replace-with-a-long-random-string-at-least-32-chars"
APP_ENCRYPTION_KEY="replace-with-a-long-random-string-at-least-32-chars"

SHOPIFY_API_KEY="your-shopify-app-api-key"
SHOPIFY_API_SECRET="your-shopify-app-api-secret"
SHOPIFY_SCOPES="read_orders,read_customers,write_customers"
SHOPIFY_WEBHOOK_BASE_URL="http://localhost:3000"
SHOPIFY_WEBHOOK_SECRET="your-shopify-app-api-secret-or-webhook-secret"

WHATSAPP_PROVIDER="META"
WHATSAPP_GRAPH_API_VERSION="v23.0"
WHATSAPP_ACCESS_TOKEN="meta-system-user-or-app-token"
WHATSAPP_PHONE_NUMBER_ID="your-phone-number-id"
WHATSAPP_BUSINESS_ACCOUNT_ID="your-waba-id"
WHATSAPP_WEBHOOK_VERIFY_TOKEN="shared-token-for-meta-verification"
META_APP_SECRET="optional-meta-app-secret-for-signature-validation"
WHATSAPP_ORDER_CONFIRMATION_TEMPLATE_NAME=""
WHATSAPP_ORDER_CONFIRMATION_TEMPLATE_LANGUAGE="en_US"
WHATSAPP_COD_CONFIRMATION_TEMPLATE_NAME=""
WHATSAPP_COD_CONFIRMATION_TEMPLATE_LANGUAGE="en_US"
```

Notes:

- `NEXT_PUBLIC_APP_URL` should match the public app URL configured in Shopify.
- `SHOPIFY_WEBHOOK_BASE_URL` should be a publicly reachable HTTPS URL outside local development.
- `SHOPIFY_WEBHOOK_SECRET` is used to validate `X-Shopify-Hmac-Sha256` against the raw request body.
- `SESSION_SECRET` and `APP_ENCRYPTION_KEY` should be rotated and stored in a secret manager in production.
- If `WHATSAPP_ORDER_CONFIRMATION_TEMPLATE_NAME` is set, order confirmations are sent as template messages; otherwise the app falls back to free-form text.
- `WHATSAPP_WEBHOOK_VERIFY_TOKEN` is required for Meta's GET webhook challenge flow.
- `META_APP_SECRET` is optional in this implementation, but if set it will validate `X-Hub-Signature-256` on inbound WhatsApp webhooks.

## Local Setup

1. Install dependencies:

```bash
npm install
```

2. Generate Prisma client:

```bash
npm run prisma:generate
```

3. Run the first migration:

```bash
npm run prisma:migrate -- --name init_phase_a
```

4. Start the app:

```bash
npm run dev
```

5. Open `http://localhost:3000`

## Shopify App Setup

This implementation uses the classic authorization-code OAuth flow for a server-rendered app. For a production embedded app, Shopify now recommends managed installation and token exchange, but this Phase A code is appropriate for the requested direct OAuth connection flow.

Configure your Shopify app with:

- App URL: `https://your-domain.com`
- Allowed redirection URL:
  - `https://your-domain.com/api/integrations/shopify/callback`

After onboarding a merchant workspace, the dashboard will start the install flow using the store's `myshopify.com` domain.

## What Phase A Does

### Authentication

- `POST /api/auth/signup`
- `POST /api/auth/login`
- `POST /api/auth/logout`

Sessions are stored in an HTTP-only signed JWT cookie.

### Merchant onboarding

- `POST /api/onboarding/account`

This creates:

- `Account`
- `AccountMembership`
- `Store`
- placeholder `WhatsAppConnection`

### Shopify connection

- `GET /api/integrations/shopify/install`
- `GET /api/integrations/shopify/callback`
- `GET /api/integrations/shopify/status`

OAuth callback handling includes:

- state validation
- HMAC validation
- access-token exchange
- shop metadata sync
- encrypted token persistence
- default webhook registration

### Webhook subscriptions registered during install

- `ORDERS_CREATE`
- `ORDERS_FULFILLED`
- `APP_UNINSTALLED`

### Phase B webhook and messaging flow

- `POST /api/integrations/shopify/webhooks/orders-created`

This route now:

- verifies Shopify webhook signatures using the raw payload
- deduplicates delivery using Shopify event ids when present
- stores webhook event records
- upserts customers, orders, and order items
- creates or reuses a WhatsApp conversation
- sends an order confirmation via WhatsApp Cloud API
- logs outbound message status and provider message ids

The outbound message path supports:

- Meta Cloud API text messages
- Meta Cloud API template messages when `WHATSAPP_ORDER_CONFIRMATION_TEMPLATE_NAME` is configured

### Phase C COD verification flow

- COD orders are detected from Shopify payment gateway names such as `cod`, `cash on delivery`, or `manual`.
- Matching orders are stored with COD verification state on the `Order` record.
- After the order confirmation, the app sends a COD prompt asking the customer to reply `YES` or `NO`.
- `GET /api/integrations/whatsapp/webhook` handles Meta's webhook verification challenge.
- `POST /api/integrations/whatsapp/webhook` ingests inbound WhatsApp messages, stores them in the conversation log, and updates the latest pending COD order for that customer.
- `YES` marks the order `CONFIRMED`; `NO` marks it `REJECTED`.
- The app sends a short acknowledgement message after the COD state changes.

### Migration generated for this phase

The Prisma migration for the current schema lives at:

```text
prisma/migrations/20260419_000001_init_phase_c/migration.sql
```

Because this workspace did not have a package manager binary available at generation time, the migration file was created manually to match the Prisma schema and then validated with the local schema-check script below.

## Local Validation Workflow

These commands use the bundled Node runtime already available in the Codex desktop environment:

```powershell
& 'C:\Users\Admin\.cache\codex-runtimes\codex-primary-runtime\dependencies\node\bin\node.exe' .\scripts\validate-schema.mjs
```

```powershell
$env:NEXT_PUBLIC_APP_URL='http://localhost:3000'
$env:WHATSAPP_WEBHOOK_VERIFY_TOKEN='test-verify-token'
$env:META_APP_SECRET='meta-secret'
$env:WHATSAPP_PHONE_NUMBER_ID='1234567890'
$env:WHATSAPP_ACCESS_TOKEN='token'
& 'C:\Users\Admin\.cache\codex-runtimes\codex-primary-runtime\dependencies\node\bin\node.exe' .\scripts\check-meta-webhook-config.mjs
```

```powershell
& 'C:\Users\Admin\.cache\codex-runtimes\codex-primary-runtime\dependencies\node\bin\node.exe' .\scripts\simulate-cod-flow.mjs
```

Or run the whole validation bundle at once:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\run-cod-local-workflow.ps1
```

### What each validation script does

- `scripts/validate-schema.mjs`
  Confirms every Prisma model and enum appears in the migration SQL and checks that all COD columns exist on the `Order` table.
- `scripts/check-meta-webhook-config.mjs`
  Verifies that the Meta webhook env vars are present and prints the exact URLs that need to be configured.
- `scripts/generate-meta-signature.mjs`
  Produces a valid `x-hub-signature-256` header for a payload file using `META_APP_SECRET`.
- `scripts/simulate-cod-flow.mjs`
  Runs an end-to-end in-memory simulation of the COD workflow, including duplicate-reply idempotency.

## Full Local Testing Workflow

### 1. Apply the migration

Once `npm` or another package manager is available in your environment:

```bash
npm install
npm run prisma:generate
npm run prisma:migrate -- --name init_phase_c
```

### 2. Start the app locally

```bash
npm run dev
```

### 3. Configure webhook URLs

- Shopify `orders/create`:
  `http://localhost:3000/api/integrations/shopify/webhooks/orders-created`
- Meta WhatsApp verification and inbound webhook:
  `http://localhost:3000/api/integrations/whatsapp/webhook`

For public testing, expose your app with a tunnel such as `ngrok` and use the HTTPS URL instead of localhost.

### 4. Simulate the Shopify COD order

Use `scripts/fixtures/shopify-cod-order.json` as your sample payload.

Expected result:

- order row is created
- `isCod = true`
- `codVerificationStatus = PENDING`
- order confirmation message is logged
- COD prompt message is logged

### 5. Simulate the customer reply

Use `scripts/fixtures/meta-reply-yes.json` as the payload and generate a Meta signature:

```powershell
$env:META_APP_SECRET='meta-secret'
& 'C:\Users\Admin\.cache\codex-runtimes\codex-primary-runtime\dependencies\node\bin\node.exe' .\scripts\generate-meta-signature.mjs .\scripts\fixtures\meta-reply-yes.json
```

Expected result:

- inbound WhatsApp message is stored
- pending COD order changes to `CONFIRMED`
- one acknowledgement message is sent
- replaying the same inbound message id does not process the confirmation again

## Simulated E2E Result

The end-to-end COD simulation currently passes with this expected sequence:

1. Shopify creates a COD order.
2. Order webhook stores the order.
3. WhatsApp order confirmation is queued.
4. WhatsApp COD verification prompt is queued.
5. Customer replies `YES`.
6. Order changes to `CONFIRMED`.
7. Duplicate replay of the same WhatsApp reply is deduplicated.

## Testing Strategy

### Unit tests

- account signup/login validation
- slug generation and duplicate store checks
- Shopify HMAC validation
- AES encryption/decryption helpers
- session cookie issue/verify lifecycle

### Integration tests

- onboarding route provisions tenant + store correctly
- Shopify callback persists token and store sync fields
- duplicate onboarding/store connection attempts are blocked
- orders/create webhook persists customer, order, and line items
- duplicate webhook deliveries do not create duplicate orders
- successful webhook processing creates an outbound WhatsApp message log
- COD orders enter `PENDING` verification and create a COD prompt message
- inbound WhatsApp webhook with `YES` marks the latest pending COD order confirmed
- inbound WhatsApp webhook with `NO` marks the latest pending COD order rejected

### End-to-end tests

- signup -> onboarding -> dashboard
- dashboard -> Shopify install redirect
- callback success -> connected Shopify state visible in dashboard
- Shopify test order webhook -> order stored -> WhatsApp confirmation sent
- COD order webhook -> COD prompt sent -> customer replies YES -> order status becomes CONFIRMED

### Manual QA checklist

- invalid `myshopify.com` domains are rejected
- sign-in fails with incorrect password
- onboarding cannot be repeated for the same user
- callback with invalid state or invalid HMAC is rejected
- successful install updates store status from `PENDING` to `ACTIVE`
- orders/create with valid HMAC stores the order snapshot
- repeated deliveries with the same Shopify event id are deduplicated
- missing phone number stores the order but skips WhatsApp send with an explanation
- COD order replies that are not `YES` or `NO` are stored but ignored for state changes
- a repeated Shopify order webhook does not reset an already confirmed COD order back to pending

## Vercel Deployment Prep

### Required environment variables in Vercel

- `DATABASE_URL`
- `NEXT_PUBLIC_APP_URL`
- `SESSION_SECRET`
- `APP_ENCRYPTION_KEY`
- `SHOPIFY_API_KEY`
- `SHOPIFY_API_SECRET`
- `SHOPIFY_SCOPES`
- `SHOPIFY_WEBHOOK_BASE_URL`
- `SHOPIFY_WEBHOOK_SECRET`
- `WHATSAPP_PROVIDER`
- `WHATSAPP_GRAPH_API_VERSION`
- `WHATSAPP_ACCESS_TOKEN`
- `WHATSAPP_PHONE_NUMBER_ID`
- `WHATSAPP_BUSINESS_ACCOUNT_ID`
- `WHATSAPP_WEBHOOK_VERIFY_TOKEN`
- `META_APP_SECRET`

Optional:

- `WHATSAPP_ORDER_CONFIRMATION_TEMPLATE_NAME`
- `WHATSAPP_ORDER_CONFIRMATION_TEMPLATE_LANGUAGE`
- `WHATSAPP_COD_CONFIRMATION_TEMPLATE_NAME`
- `WHATSAPP_COD_CONFIRMATION_TEMPLATE_LANGUAGE`

### Vercel-specific checklist

- Set `NEXT_PUBLIC_APP_URL` and `SHOPIFY_WEBHOOK_BASE_URL` to your production Vercel URL.
- Add `https://your-domain.com/api/integrations/shopify/callback` to Shopify allowed redirects.
- Register Shopify webhook endpoints against the same production domain.
- In Meta, configure the webhook callback URL as:
  `https://your-domain.com/api/integrations/whatsapp/webhook`
- Use the same `WHATSAPP_WEBHOOK_VERIFY_TOKEN` in Meta and in Vercel env vars.
- If you enable `META_APP_SECRET`, Meta webhook signature verification is enforced on every POST.
- Run the Prisma migration against the production database before switching traffic.
- Keep `postinstall: prisma generate` enabled so Vercel builds have a generated client.

### Recommended production hardening before launch

- Add a real queue for outbound WhatsApp sends and webhook processing retries.
- Add structured logging around webhook ids and order ids.
- Add alerting for webhook failures and message-send failures.
- Add a merchant UI surface for reviewing `REJECTED` COD orders.

## Production Notes

- Move webhook processing to async jobs in Phase B.
- Replace local secrets with a managed secret store.
- Add request logging, rate limiting, and audit trails before production launch.
- Add a real WhatsApp credential flow in the next implementation step.
