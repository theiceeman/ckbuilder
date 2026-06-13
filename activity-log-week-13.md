# Activity Log - Week 13

## Summary
Week 13 delivered two major platform features that together form a complete **merchant and developer growth engine** for the CKB Nervos ecosystem. The **AI Shop Builder** enables any merchant to launch a fully functional e-commerce store — powered by WT Payments with native CKB support — in minutes, using only a conversational AI agent. The **Developer Platform** (API Keys + Webhooks) turns WT Payments into programmable infrastructure that any developer can embed into their own applications, multiplying the number of places where CKB can be spent across the ecosystem.

---

## Features Built This Week

### 1. AI Shop Builder

#### What It Does
Merchants can create a branded e-commerce website hosted on a subdomain (`{businessname}.yourdomain.com`) without writing a single line of code. An AI agent guides them through setup conversationally and remembers everything about their shop across sessions.

#### Endpoints Built

| Endpoint | Method | Description |
|---|---|---|
| `/user/shop` | `POST` | Create a shop (sets subdomain, currency, business name) |
| `/user/shop` | `PUT` | Update shop (e.g. publish: `{ "status": "published" }`) |
| `/user/shop/ai/chat` | `POST` | Send message to AI agent — auto-applies theme changes |
| `/user/shop/ai/history` | `GET` | Full conversation history (no internal reasoning data) |
| `/user/shop/ai/memory` | `DELETE` | Reset memory — start a fresh design session |
| `/user/shop/products` | `POST` | Add a product (name, price, description, stock, category) |
| `/user/shop/products/:id/images` | `POST` | Upload up to 5 product images (multipart) |

#### AI Agent Setup Flow (Example)
```
POST /user/shop
{ "business_name": "Adaeze Fabrics", "subdomain": "adaeze-fabrics", "currency": "NGN" }
→ Shop created at https://adaeze-fabrics.yourdomain.com

POST /user/shop/ai/chat
{ "message": "Help me design a colorful Ankara fashion store with a hero section and a grid layout" }
→ AI responds with theme_config JSON → automatically applied to shop

POST /user/shop/products
{ "name": "Premium Ankara Fabric - 6 yards", "price": 15000, "stock": 50, "track_stock": true }

PUT /user/shop
{ "status": "published" }
→ Shop is live. Every checkout supports fiat and CKB payment options.
```

#### AI Agent Memory Architecture
- **Persistent multi-turn memory** — conversations stored in `ai_shop_conversations` table with full message history including `reasoning_details` (Claude).
- The agent remembers prior sessions, knows current products and prices, and continues reasoning from where it left off.
- System prompt is **dynamically rebuilt** on every request with live shop and product data.
- **Primary model:** `anthropic/claude-haiku-latest` via OpenRouter (reasoning enabled).
- **Fallback model:** `meta-llama/llama-3.3-70b-instruct:free` (free tier, no reasoning) — ensures the feature never goes down if primary quota is exhausted.

---

### 2. Developer Platform — API Keys

#### What It Does
Every merchant gets two key pairs (TEST and LIVE), allowing developers to integrate WT Payments — including CKB payments — into any third-party application using standard secret/public key authentication.

| Key | Format | Use |
|---|---|---|
| Secret key | `sk_live_<40 hex chars>` | Server-side API calls — never expose publicly |
| Public key | `pk_live_<40 hex chars>` | Client-side payment widget initialization |

#### Endpoints Built

| Endpoint | Method | Description |
|---|---|---|
| `/api/user/settings/api-key` | `POST` | Generate a key pair — private key shown **once**, then hashed in DB |
| `/api/user/settings/general` | `POST` | Switch environment: `{ "current_environment": "LIVE" }` |
| `/api/user/settings/api-key/verify` | `POST` | Verify a secret key without dashboard login |

---

### 3. Developer Platform — Webhooks

#### What It Does
Real-time event delivery to developer servers. When a payment is confirmed on-chain (including CKB transactions), the developer's registered URL receives an immediate POST with a signed payload.

#### Endpoints Built

| Endpoint | Method | Description |
|---|---|---|
| `/api/user/settings/webhook` | `POST` | Save webhook URL (HTTPS only) |
| `/api/user/settings/webhook/secret/generate` | `POST` | Generate signing secret — shown **once** |
| `/api/user/settings/webhook/verify` | `POST` | Test endpoint reachability (sends test `payment.confirmed` event) |
| `/api/user/settings/webhook/logs` | `GET` | Delivery logs with HTTP status codes and error messages |

#### Webhook Events

| Event | Trigger |
|---|---|
| `payment.confirmed` | Crypto payment confirmed on-chain (EVM or CKB) |
| `payment.failed` | Payment attempt failed or expired |
| `payment.pending` | Payment created, awaiting confirmation |
| `payout.completed` | Fiat or crypto payout successfully sent |
| `payout.failed` | Payout attempt failed |

#### Security — HMAC-SHA256 Signature Verification
Every webhook POST is signed. The signature is in the `X-WT-Signature` header as `sha256=<hex>`. Merchants verify using their unique signing secret with constant-time comparison to prevent timing attacks.

**Retry policy:** 3 attempts with exponential back-off (1s → 2s → 4s). Failed deliveries are logged and visible in the dashboard.

#### Sample Payload
```json
{
  "event": "payment.confirmed",
  "environment": "LIVE",
  "timestamp": "2026-06-13T10:45:00.000Z",
  "data": {
    "paymentId": "d0f9ba1e-...",
    "businessReferenceId": "order_12345",
    "amount": 50000,
    "currency": "NGN",
    "transactionHash": "0xabc123...",
    "confirmedAt": "2026-06-13T10:44:58.000Z"
  }
}
```

---

## Database Tables Added

| Table | Purpose |
|---|---|
| `shops` | Merchant shop config, subdomain, theme, publish status |
| `shop_products` | Product catalog with images, variants, and stock tracking |
| `ai_shop_conversations` | Full AI conversation history per shop (with `reasoning_details`) |
| `webhook_logs` | Every webhook delivery attempt, HTTP response, and retry record |
| `business_settings.webhook_signing_secret` | Per-merchant HMAC signing secret (new column) |

---

## Environment Variables Added

```env
# AI Shop Builder
OPENROUTER_API_KEY=sk-or-...
OPENROUTER_PRIMARY_MODEL=anthropic/claude-haiku-latest
OPENROUTER_FALLBACK_MODEL=meta-llama/llama-3.3-70b-instruct:free
SHOP_BASE_DOMAIN=yourdomain.com

# Webhooks
WEBHOOK_SECRET=global-fallback-secret
APP_ENV=production
```

---

## How These Features Grow the CKB Nervos Ecosystem

### The Problem
CKB has limited real-world merchant adoption. Most merchants don't accept CKB because there is no easy checkout experience, no simple developer integration path, and no ready-made e-commerce tooling built around it.

### AI Shop Builder → Direct CKB Merchant Adoption
Every shop created through our AI builder accepts payments via WT Payments, which has CKB integrated at the infrastructure level (`CKBService`, wallet generation, balance polling). A merchant who has never heard of CKB now accepts it automatically — their customers who hold CKB can spend it at real stores for real goods. This is a **bottom-up adoption strategy**: merchants don't choose CKB, they get it by default.

### Developer Platform → CKB Embedded in Third-Party Apps
The API key + webhook system turns WT Payments into programmable infrastructure. Any developer can embed CKB payment acceptance into their existing app without understanding blockchain internals. A single developer building a popular marketplace brings CKB to all of their users. This is a **network effects strategy**.

### Combined Flywheel
```
More shops built via AI → More places to spend CKB
         ↓
More developers integrate webhooks → More apps accept CKB
         ↓
More CKB transactions on-chain → More ecosystem activity
         ↓
More visibility for CKB → More users and merchants
         ↓
More shops built via AI → (repeat)
```

### CKB Technical Integration (`app/Services/CKBService.ts`)
Built on `@ckb-lumos/lumos`:
- **Wallet generation** — deterministic key derivation via `hd.key`, SECP256K1_BLAKE160 lock scripts
- **Address generation** — testnet (AGGRON4) and mainnet compatible
- **Balance queries** — CKB Indexer (cell collector)
- **Transaction lookup** — CKB RPC
- **Block queries** — chain tip and block-by-number
- Network RPC URL loaded from `crypto_networks` DB table (`chainKey = 'ckb'`) — switching testnet → mainnet requires only a DB update, no code change.
