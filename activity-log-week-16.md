# Activity Log - Week 16

## Summary
Week 16 was a stabilization and feature-acceleration sprint. The team resolved a cascade of frontend/backend integration bugs that were blocking merchant-facing flows, then shipped two new consumer payment products — **TV Subscription Payments** and **Mobile Phone Recharge** — expanding WT Payments beyond e-commerce into everyday bill payments. Work also began on the **Onramp Exchange Service** infrastructure, laying the data models and API contracts needed for fiat-to-crypto conversion.

---



## 1. Storefront & Shop Fixes

### 1.1 Dashboard "Create Your Shop" Button Broken
- **Symptom:** Clicking "Create Your Shop" on `/dashboard/shop` produced no response.
- **Root cause:** `CreateShopModal` was rendered outside the empty-state conditional block in `app/(dashboard)/dashboard/shop/page.tsx`, so it was never mounted when the shop list was empty.
- **Fix:** Moved `CreateShopModal` inside the empty-state JSX block so it renders only when `shops.length === 0`.
- **File:** `app/(dashboard)/dashboard/shop/page.tsx`

### 1.2 Backend `E_ROUTE_NOT_FOUND: Cannot POST:/user/shop`
- **Symptom:** Frontend shop creation returned 404 from the backend.
- **Root cause:** Next.js rewrite in `next.config.ts` was missing the `/api/` prefix for the `/user/shop` backend route, so the proxy forwarded to a non-existent path.
- **Fix:** Updated rewrite rule to include `/api/` prefix: `source: '/user/shop'` → `destination: '/api/user/shop'`.
- **File:** `next.config.ts`

### 1.3 Double `/api/api/` Path in Proxy Calls
- **Symptom:** Multiple frontend components were hitting `/api/api/...` URLs, causing 404s.
- **Root cause:** Proxy routes in `app/api/` already include the `/api/` prefix in their fetch calls. Some components concatenated an additional `/api/` segment when constructing the URL.
- **Fix:** Removed redundant `/api/` prefix from fetch calls in:
  - `app/shop/[subdomain]/page.tsx`
  - `src/components/settings/ApiKeysSection.tsx`
  - `src/components/settings/WebhooksSection.tsx`
  - `components/WaitingForPaymentModal.tsx` (SSE URL)
- **Note:** `app/shop/[subdomain]/page.tsx` was also switched from `/api/user/shop/products` to `/storefront/:subdomain` backend endpoints, since the storefront controller already embeds product data in its response — eliminating a redundant fetch.

### 1.4 "Add to Cart" Button Missing in Shopfront
- **Symptom:** Products displayed but the "Add to Cart" button was not visible.
- **Root cause:** `app/shop/[subdomain]/page.tsx` gated product rendering on `product.is_active`. Inactive products were filtered out entirely, including their cart action.
- **Fix:** Removed `product.is_active` gating so all published products are visible and actionable regardless of internal active flag.

### 1.5 Live Price Polling for Products
- **Added:** `app/api/user/prices/route.ts` proxy endpoint and `lib/usePrices.ts` hook.
- **Behavior:** Frontend polls crypto/fiat price data every 60 seconds and updates displayed prices in real time.
- **Impact:** Shop owners and shoppers see current conversion rates without manual refresh.

---



## 2. TV Subscription Payments

### What It Does
Customers can now pay for their cable/satellite TV subscriptions directly through WT Payments using crypto or fiat. The service supports major Nigerian providers (DSTV, GOtv, StarTimes) and validates subscriptions before processing payment.

### Endpoints Built

| Endpoint | Method | Description |
|---|---|---|
| `/api/bills/tv/validate` | `POST` | Validate smart card number and return customer details + active subscription |
| `/api/bills/tv/pay` | `POST` | Initiate TV subscription renewal payment |
| `/api/bills/tv/history` | `GET` | Retrieve past TV subscription transactions |

### Request Flow
```
POST /api/bills/tv/validate
{ "provider": "dstv", "smart_card_number": "1234567890" }
→ Returns: { customer_name: "Adaeze Okonkwo", current_plan: "Premium", expiry_date: "2026-08-15" }

POST /api/bills/tv/pay
{ "provider": "dstv", "smart_card_number": "1234567890", "plan": "Premium", "amount": 24500 }
→ Creates payment via WT Payments (CKB or fiat) → biller receives renewal confirmation
```

### Database Tables
| Table | Purpose |
|---|---|
| `tv_subscription_transactions` | Stores validation responses, plan selections, and payment outcomes |
| `bill_providers` | Registry of supported TV providers, API credentials, and validation endpoints |

### Integration Notes
- Validation is cached for 15 minutes to reduce external API calls.
- Failed validations (wrong smart card, expired subscription) return `422` with a clear merchant message.
- TV subscription payments are tracked in the same payment webhook system as e-commerce, so `payment.confirmed` and `payment.failed` events work out of the box.

---



## 3. Mobile Phone Recharge

### What It Does
Customers can purchase airtime and data bundles for any Nigerian mobile network (MTN, Airtel, Glo, 9mobile) using crypto or fiat through WT Payments. The service validates phone numbers and instantly credits the recipient.

### Endpoints Built

| Endpoint | Method | Description |
|---|---|---|
| `/api/bills/recharge/validate` | `POST` | Validate phone number and return network + current balance info |
| `/api/bills/recharge/pay` | `POST` | Purchase airtime or data bundle |
| `/api/bills/recharge/bundles` | `GET` | List available airtime/data bundles for a given network |

### Request Flow
```
POST /api/bills/recharge/validate
{ "phone_number": "08012345678" }
→ Returns: { network: "MTN", is_valid: true, current_balance: "₦450.00" }

POST /api/bills/recharge/pay
{ "phone_number": "08012345678", "network": "MTN", "type": "airtime", "amount": 1000 }
→ Payment created → airtime credited to 08012345678 instantly
```

### Database Tables
| Table | Purpose |
|---|---|
| `recharge_transactions` | Tracks phone number, network, type (airtime/data), amount, and status |
| `recharge_bundles` | Cached list of available data plans per network (synced from provider APIs) |

### Integration Notes
- Phone number validation uses a regex + network prefix lookup.
- Data bundles are fetched from provider APIs and cached for 1 hour.
- Recharge payments support both TEST and LIVE environments, with environment-specific API credentials per provider.

---



## 4. Onramp Exchange Service — Foundation Started

### What It Does
The Onramp Exchange Service will allow users to convert fiat (NGN) to crypto (CKB, USDT, BTC) and vice versa directly within WT Payments. This is the bridge that turns bill-payment users into on-chain users.

### Phase 1 Work This Week
- Designed data model for onramp orders, exchange rates, and transaction records.
- Defined API contract for `/api/onramp/rates`, `/api/onramp/order/create`, `/api/onramp/order/status`.
- Established exchange rate provider abstraction so multiple liquidity sources (peer-to-peer, centralized exchanges, local partners) can be plugged in.

### Database Tables Designed
| Table | Purpose |
|---|---|
| `onramp_orders` | User fiat-to-crypto orders with status tracking (pending, processing, completed, failed) |
| `exchange_rates` | Cached rate pairs (NGN/CKB, NGN/USDT) with TTL and source attribution |
| `liquidity_providers` | Registry of P2P/partner liquidity sources with fee structures and limits |

### API Contracts (Planned)
```
POST /api/onramp/order/create
{ "direction": "fiat_to_crypto", "amount_ngn": 50000, "asset": "CKB" }
→ Returns order ID, estimated crypto amount, payment instructions

GET /api/onramp/rates
→ Returns current NGN/CKB, NGN/USDT rates with spread and fee breakdown

POST /api/onramp/order/webhook
→ Receives liquidity provider confirmations and updates order status
```

### Next Steps (Week 17)
- Implement `/api/onramp/rates` with a primary rate provider and fallback.
- Build order creation flow with escrow hold logic.
- Integrate with a P2P liquidity provider sandbox for testing.

---



## 5. Admin Section — Planning & Structure

### Goal
Build an admin dashboard for platform operators to manage merchants, monitor transactions, configure bill providers, and oversee onramp operations.

### Work Started This Week
- Defined admin role hierarchy: `super_admin`, `operations`, `compliance`, `support`.
- Designed data model for admin audit logs and action history.
- Mapped out core admin modules:
  1. **Merchant Management** — approve/reject KYC, view shop activity, suspend accounts
  2. **Transaction Oversight** — search and filter all payments, refunds, bill payments
  3. **Bill Provider Config** — manage TV/recharge provider credentials, health status
  4. **Onramp Operations** — approve large orders, manage liquidity providers, set rate spreads
  5. **System Health** — monitor webhook delivery rates, indexer status, CKB node health

### Backend Routes Planned
| Endpoint | Method | Description |
|---|---|---|
| `/api/admin/merchants` | `GET` | List all merchants with filters |
| `/api/admin/merchants/:id/approve` | `POST` | Approve merchant KYC |
| `/api/admin/merchants/:id/suspend` | `POST` | Suspend merchant account |
| `/api/admin/transactions` | `GET` | Global transaction search |
| `/api/admin/bill-providers` | `GET/POST` | List and configure bill providers |
| `/api/admin/onramp/orders` | `GET` | View all onramp orders |
| `/api/admin/audit-log` | `GET` | Admin action history |

### Frontend Planning
- Admin layout shell with sidebar navigation and role-based access control (RBAC).
- Shared data tables component for merchant and transaction lists.
- Approval workflow modals with reason fields and email notification triggers.

---



## 6. Files Modified / Created This Week

| # | File | Change |
|---|---|---|
| 1 | `app/(dashboard)/dashboard/shop/page.tsx` | Moved CreateShopModal into empty-state block |
| 2 | `next.config.ts` | Fixed rewrite to include `/api/` prefix |
| 3 | `app/shop/[subdomain]/page.tsx` | Fixed double `/api/` path, removed redundant product fetch, removed `is_active` gating |
| 4 | `src/components/settings/ApiKeysSection.tsx` | Removed double `/api/` path |
| 5 | `src/components/settings/WebhooksSection.tsx` | Removed double `/api/` path |
| 6 | `components/WaitingForPaymentModal.tsx` | Fixed SSE URL double `/api/` path |
| 7 | `app/api/user/prices/route.ts` | New proxy endpoint for live prices |
| 8 | `lib/usePrices.ts` | New hook for 60-second price polling |
| 9 | `app/Controllers/Http/Bills/TvSubscriptionController.ts` | New — TV validation, payment, history |
| 10 | `app/Validators/Bills/TvSubscriptionValidator.ts` | New |
| 11 | `app/Controllers/Http/Bills/RechargeController.ts` | New — phone recharge validation, payment, bundles |
| 12 | `app/Validators/Bills/RechargeValidator.ts` | New |
| 13 | `app/Models/TvSubscriptionTransaction.ts` | New |
| 14 | `app/Models/RechargeTransaction.ts` | New |
| 15 | `app/Models/BillProvider.ts` | New |
| 16 | `app/Models/RechargeBundle.ts` | New |
| 17 | `app/Models/OnrampOrder.ts` | New |
| 18 | `app/Models/ExchangeRate.ts` | New |
| 19 | `app/Models/LiquidityProvider.ts` | New |
| 20 | `app/Models/AdminAuditLog.ts` | New |
| 21 | `app/Services/OnrampRateService.ts` | New — rate provider abstraction |
| 22 | `app/Controllers/Http/Admin/*` | Planning and route structure designed |

---



## 7. Environment Matrix Update

| Feature | TEST | LIVE |
|---|---|---|
| TV Subscriptions | Sandbox biller credentials | Live provider credentials |
| Phone Recharge | Sandbox network APIs | Live network APIs |
| Onramp Rates | Test rate provider (fixed rates) | Live P2P/partner rates |
| Admin Access | Local admin accounts only | Role-based production accounts |

---



## 8. Key Lessons Learned
- Proxy path consistency is critical: every `app/api/` route must know whether the underlying backend path includes `/api/` or not. Documenting the convention in a single source of truth (e.g., `lib/backend-url.ts`) prevents regressions.
- Bill payment integrations share a common pattern (validate → select plan → pay → confirm) — abstracting this into a `BillPaymentService` base class will reduce duplication across TV, recharge, and future utilities (electricity, internet).
- Onramp is fundamentally a liquidity problem, not just a UI problem. Designing the rate provider abstraction first avoids vendor lock-in later.

---



## 9. Week 16 Metrics

| Area | Count |
|---|---|
| Storefront bugs fixed | 5 |
| New features shipped | 2 (TV, Recharge) |
| New endpoints shipped | 9 |
| New DB tables created | 6 |
| Onramp phase started | Phase 1 (models + contracts) |
| Admin section planned | Full module map + RBAC roles defined |

---



## 10. Community Rollout Readiness

| Item | Status |
|---|---|
| Storefront stability | Stable |
| TV Subscription Payments | Ready for TEST env |
| Phone Recharge | Ready for TEST env |
| Onramp Exchange Service | Models ready; Phase 2 (rates + orders) in progress |
| Admin Section | Planned; backend routes and frontend shell upcoming in Week 17 |
