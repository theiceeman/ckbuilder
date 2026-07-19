# Activity Log - Week 17

## Summary
Week 17 completed the two major platform expansions started in Week 16: the **Onramp Exchange Service** (Phase 2 — rates, order creation, and fiat-to-crypto conversion flow) and the **Admin Section** (backend routes, frontend dashboard shell, and RBAC). Both features move WT Payments toward a full-stack financial platform: onramp turns bill-pay users into crypto users, and the admin section gives operators the tooling to manage a live payments business.

---



## 1. Onramp Exchange Service — Phase 2 Implementation

### What It Does
Users can now convert NGN fiat to CKB, USDT, or BTC directly within WT Payments. The service fetches live exchange rates from a primary liquidity provider with automatic failover, creates escrow-backed orders, and tracks the full conversion lifecycle.

### Endpoints Built

| Endpoint | Method | Description |
|---|---|---|
| `/api/onramp/rates` | `GET` | Returns current NGN/CKB, NGN/USDT, NGN/BTC rates with spread and fee breakdown |
| `/api/onramp/order/create` | `POST` | Create a fiat-to-crypto order; returns payment instructions and order ID |
| `/api/onramp/order/status/:id` | `GET` | Poll order status (pending, processing, completed, failed) |
| `/api/onramp/order/webhook` | `POST` | Receives liquidity provider confirmations and updates order status |
| `/api/onramp/order/history` | `GET` | User's past onramp orders |

### Request Flow
```
GET /api/onramp/rates
→ { "CKB": { rate: 0.85, spread: 0.02, fee_ngn: 150 }, "USDT": { rate: 850.00, spread: 0.01, fee_ngn: 100 } }

POST /api/onramp/order/create
{ "direction": "fiat_to_crypto", "amount_ngn": 50000, "asset": "CKB" }
→ { order_id: "onr_abc123", estimated_crypto: 58500, payment_reference: "WT-ONR-001", instructions: "Transfer ₦50,000 to GTBank 0123456789" }

POST /api/onramp/order/webhook
{ order_id: "onr_abc123", status: "completed", tx_hash: "0xdef456..." }
→ Order marked completed → crypto credited to user's WT Payments wallet
```

### Rate Provider Architecture
- **Primary:** P2P liquidity provider API (aggregated order book).
- **Fallback:** Secondary provider or cached rate (max 5 minutes old).
- **TTL:** Rates cached in `exchange_rates` table with 30-second sliding window.
- **Spread model:** 1–2% spread applied automatically; configurable per `liquidity_provider`.

### Database Tables Created (Week 15) + Populated (Week 17)
| Table | Status |
|---|---|
| `onramp_orders` | Created — populated with status tracking fields and escrow hold logic |
| `exchange_rates` | Created — TTL-based caching implemented |
| `liquidity_providers` | Created — two providers registered (primary + fallback) |

### Order Lifecycle States
```
pending → processing → completed
         ↓
       failed (with reason: insufficient_liquidity, provider_timeout, user_timeout)
```

### Security
- Onramp orders require a valid `current_environment` session (TEST or LIVE).
- Webhook endpoint validates `X-WT-Signature` using the same HMAC-SHA256 scheme as payment webhooks.
- Large orders (> ₦500,000 LIVE, > ₦100,000 TEST) flag for admin review via audit log.

### Bug Fixes During Implementation
- **Race condition in rate cache:** Concurrent requests for `/api/onramp/rates` could insert stale rates. Fixed by adding a database-level unique constraint on `(base_currency, quote_currency, fetched_at)` and using `upsert`.
- **Webhook replay:** Onramp webhook endpoint was idempotent but not replay-safe. Added `order_id + event_id` deduplication key.
- **Escrow hold timeout:** Orders left in `processing` for > 15 minutes (TEST) or > 1 hour (LIVE) now auto-cancel and release the hold.

---



## 2. Admin Section — Built

### What It Does
The Admin Section gives platform operators full visibility and control over merchants, transactions, bill providers, and onramp operations. It includes role-based access control (RBAC), audit logging, and a responsive dashboard shell.

### Backend Routes Implemented

| Endpoint | Method | Description |
|---|---|---|
| `/api/admin/merchants` | `GET` | List all merchants with search, filter by status/KYC state, and pagination |
| `/api/admin/merchants/:id/approve` | `POST` | Approve merchant KYC; triggers welcome email |
| `/api/admin/merchants/:id/reject` | `POST` | Reject KYC with reason; notifies merchant |
| `/api/admin/merchants/:id/suspend` | `POST` | Suspend merchant account and all active shops |
| `/api/admin/merchants/:id/reactivate` | `POST` | Reactivate suspended account |
| `/api/admin/transactions` | `GET` | Global transaction search with filters (type, status, date range, merchant) |
| `/api/admin/bill-providers` | `GET/POST/PATCH` | List, configure, and toggle bill providers (TV, recharge) |
| `/api/admin/bill-providers/:id/health` | `GET` | Check provider API health and latency |
| `/api/admin/onramp/orders` | `GET` | View all onramp orders with filtering by status and liquidity provider |
| `/api/admin/onramp/orders/:id/manual-resolve` | `POST` | Manually resolve a stuck onramp order |
| `/api/admin/audit-log` | `GET` | Admin action history with actor, action, target, and timestamp |
| `/api/admin/system/health` | `GET` | Aggregate health: webhook delivery rate, indexer tip lag, CKB node status |

### RBAC Roles
| Role | Permissions |
|---|---|
| `super_admin` | Full access — can manage other admins, change system settings |
| `operations` | Merchant approve/reject/suspend, transaction search, bill provider config |
| `compliance` | Read-only transaction and merchant access, audit log viewer |
| `support` | Merchant search (read-only), basic transaction lookup |

### Middleware
- `AdminAuthMiddleware` — validates admin JWT and attaches `admin.role` to request.
- `RequireRoleMiddleware` — accepts a role list and rejects requests from lower-privilege admins.
- Admin session tokens are separate from merchant/user tokens; stored in `admin_sessions` table with IP and user-agent.

### Frontend — Admin Dashboard Shell

### Structure
```
app/(admin)/admin/
├── layout.tsx          # Admin shell with sidebar, RBAC-aware nav, logout
├── page.tsx            # Dashboard home — system health cards, recent alerts
├── merchants/
│   ├── page.tsx        # Merchant table with filters and bulk actions
│   └── [id]/
│       ├── approve.tsx
│       ├── reject.tsx
│       └── suspend.tsx
├── transactions/
│   └── page.tsx        # Global transaction search with date range, export CSV
├── bill-providers/
│   └── page.tsx        # Provider cards with health status and config modals
├── onramp/
│   └── page.tsx        # Onramp order queue with manual-resolve action
└── audit-log/
    └── page.tsx        # Chronological admin action log with actor and target
```

### Components Built
| Component | Description |
|---|---|
| `AdminSidebar` | Role-aware navigation — hides `super_admin` items for lower roles |
| `AdminDataTable` | Shared table with search, sort, pagination, and row actions |
| `HealthCard` | Small metric card (value + trend + status indicator) for system health |
| `ApproveModal` | Approval form with optional note field |
| `AuditLogRow` | Displays actor, action, target type, and timestamp |

### Audit Log Design
Every admin action writes to `admin_audit_logs`:
```ts
{
  admin_id: "adm_123",
  admin_email: "ops@wtpayments.com",
  role: "operations",
  action: "approve_kyc",
  target_type: "merchant",
  target_id: "usr_456",
  metadata: { previous_status: "pending", new_status: "approved" },
  ip_address: "102.89.xx.xx",
  user_agent: "Mozilla/5.0...",
  created_at: "2026-07-14T09:30:00Z"
}
```

---



## 3. Bug Fixes & Stabilization

### 3.1 TV Validation Cache Poisoning
- **Symptom:** After a provider API returned an error, subsequent validations returned the cached error for 15 minutes.
- **Root cause:** Cache write happened before response validation; error responses were cached identically to success responses.
- **Fix:** Added `isSuccess` flag check before writing to cache; error responses are never cached.

### 3.2 Recharge Bundle Expiry
- **Symptom:** Displayed data bundles were from 24+ hours ago despite a 1-hour cache TTL.
- **Root cause:** TTL was set in milliseconds but compared against seconds in the query, making effective TTL 1/1000th of intended.
- **Fix:** Normalized TTL handling — stored as Unix timestamp in seconds, compared consistently.

### 3.3 Admin Merchant List Pagination
- **Symptom:** Page 2+ of merchant list returned the same first-page results.
- **Root cause:** Offset was being applied as a string in the query builder instead of a number, causing the DB to treat it as `0`.
- **Fix:** Explicitly cast `page` query param to `Number()` before building the offset.

### 3.4 Onramp Order Status Stuck
- **Symptom:** Orders remained in `processing` state after provider webhook confirmed completion.
- **Root cause:** Webhook handler matched on `order_id` but the provider used a different casing (`ONR_ABC123` vs `onr_abc123`).
- **Fix:** Normalized `order_id` to lowercase at ingestion time in both order creation and webhook handler.

---



## 4. Database Tables Added

| Table | Purpose |
|---|---|
| `onramp_orders` | Fiat-to-crypto orders with escrow hold, status, and provider reference |
| `exchange_rates` | Cached rate pairs with TTL and source attribution |
| `liquidity_providers` | P2P/partner liquidity sources with fee structures and limits |
| `admin_sessions` | Admin JWT sessions with IP and user-agent tracking |
| `admin_audit_logs` | Every admin action with actor, target, metadata, and timestamp |
| `recharge_transactions` | Phone recharge records with network, type, and status |
| `recharge_bundles` | Cached data plan catalog per network |
| `tv_subscription_transactions` | TV subscription payments and validation responses |
| `bill_providers` | TV and recharge provider credentials and health status |

---



## 5. Environment Variables Added

```env
# Onramp Exchange
ONRAMP_PRIMARY_PROVIDER_API_KEY=...
ONRAMP_PRIMARY_PROVIDER_BASE_URL=https://...
ONRAMP_FALLBACK_PROVIDER_API_KEY=...
ONRAMP_FALLBACK_PROVIDER_BASE_URL=https://...
ONRAMP_AUTO_RESOLVE_MINUTES_LIVE=60
ONRAMP_AUTO_RESOLVE_MINUTES_TEST=15

# Bill Payments
TV_PROVIDER_API_KEY=...
TV_PROVIDER_BASE_URL=https://...
RECHARGE_PROVIDER_API_KEY=...
RECHARGE_PROVIDER_BASE_URL=https://...

# Admin
ADMIN_JWT_SECRET=...
ADMIN_SESSION_TTL_HOURS=12
```

---



## 6. Files Modified / Created This Week

| # | File | Change |
|---|---|---|
| 1 | `app/Controllers/Http/Onramp/*` | New — rates, order create, status, webhook, history controllers |
| 2 | `app/Validators/Onramp/*` | New — order create, webhook validators |
| 3 | `app/Services/OnrampRateService.ts` | New — rate fetching with primary + fallback |
| 4 | `app/Services/OnrampOrderService.ts` | New — order lifecycle and escrow logic |
| 5 | `app/Services/BillPaymentService.ts` | New — shared base for TV and recharge |
| 6 | `app/Middleware/AdminAuthMiddleware.ts` | New — admin JWT validation |
| 7 | `app/Middleware/RequireRoleMiddleware.ts` | New — RBAC role enforcement |
| 8 | `app/Controllers/Http/Admin/*` | New — all admin route controllers |
| 9 | `app/Validators/Admin/*` | New — admin action validators |
| 10 | `app/(admin)/admin/**/*` | New — full admin dashboard shell and pages |
| 11 | `components/admin/**` | New — shared admin components (Sidebar, DataTable, HealthCard, modals) |
| 12 | `app/Models/OnrampOrder.ts` | New |
| 13 | `app/Models/ExchangeRate.ts` | New |
| 14 | `app/Models/LiquidityProvider.ts` | New |
| 15 | `app/Models/AdminSession.ts` | New |
| 16 | `app/Models/AdminAuditLog.ts` | New |
| 17 | `app/Models/RechargeTransaction.ts` | New |
| 18 | `app/Models/RechargeBundle.ts` | New |
| 19 | `app/Models/TvSubscriptionTransaction.ts` | New |
| 20 | `app/Models/BillProvider.ts` | New |

---



## 7. Week 17 Metrics

| Area | Count |
|---|---|
| Onramp endpoints shipped | 5 |
| Admin backend routes shipped | 12 |
| Admin frontend pages built | 6 |
| New DB tables created | 9 |
| Bugs fixed | 4 |
| RBAC roles defined | 4 |

---



## 8. Key Lessons Learned
- Onramp is a liquidity problem first, UX problem second. Getting the rate provider abstraction and fallback logic right prevents painful re-architecture later.
- Admin tooling requires the same rigor as user-facing features — audit logging should be retrofitted before launch, not after an incident.
- Shared abstractions (e.g., `BillPaymentService` base class) pay off immediately when the second bill type is built; the third is nearly free.
- Caching in financial flows needs explicit success/failure semantics — caching errors is worse than not caching at all.

---



## 9. Community Rollout Readiness

| Item | Status |
|---|---|
| TV Subscription Payments | Ready for community TEST |
| Mobile Phone Recharge | Ready for community TEST |
| Onramp Exchange Service | Phase 2 complete; Phase 3 (live P2P integration) in staging |
| Admin Section | Functional in TEST; ready for operations team onboarding |
| Community Testing Window | Open — awaiting backend hosting from Week 14 |
