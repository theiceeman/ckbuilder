# Activity Log - Week 19

## Summary
Week 19 focused on three major workstreams: **Admin Section enhancements and stabilization**, **AI Paid Shop Builder feature completion**, and **Cron job deployment and monitoring on Render**. The team addressed production-readiness gaps in admin tooling, completed the paid shop builder subscription and billing flow, and set up scheduled background jobs for indexer and webhook maintenance.

---



## 1. Admin Section — Enhancements & Stabilization

### What Was Done
The Admin Section, built in Week 17, received a stabilization pass and new operational capabilities based on early operator feedback.

### New Endpoints Added

| Endpoint | Method | Description |
|---|---|---|
| `/api/admin/merchants/:id/kyc` | `GET` | Fetch full KYC document details for a merchant |
| `/api/admin/merchants/:id/kyc/approve` | `POST` | Approve KYC with document verification note |
| `/api/admin/merchants/:id/kyc/reject` | `POST` | Reject KYC with reason and optional resubmission request |
| `/api/admin/shops/:id/force-publish` | `POST` | Override shop publish status (for compliance review) |
| `/api/admin/shops/:id/force-unpublish` | `POST` | Take a shop offline pending investigation |
| `/api/admin/system/config` | `GET/PATCH` | View and update platform-wide settings (fees, limits, feature flags) |
| `/api/admin/system/config/rollback` | `POST` | Rollback a config change to a previous version |

### KYC Document Workflow
- Admins can now view uploaded KYC documents (CAC certificate, government ID, proof of address) directly from the admin dashboard.
- Approve/reject actions trigger email notifications to the merchant with clear next steps.
- Rejected KYC submissions are logged with the rejection reason for audit trail.

### System Configuration Panel
- Introduced a `platform_settings` table to store key-value configuration with version history.
- Settings include: platform fee percentage, minimum payout amount, max onramp order size, feature flags (e.g., `enable_guest_checkout`, `enable_onramp`).
- Every config change is recorded in `admin_audit_logs` with old value, new value, and admin ID.

### Bug Fixes
- **Admin transaction list timezone drift:** Timestamps were stored in UTC but displayed without timezone conversion, causing confusion for Nigerian operators. Fixed by adding explicit UTC+1 conversion in the frontend `AdminDataTable` component.
- **RBAC bypass on merchant detail:** The `/api/admin/merchants/:id` detail endpoint was accessible to `support` role despite the RBAC spec. Fixed by adding `RequireRole(['super_admin', 'operations', 'compliance'])` middleware.
- **Audit log pagination missing:** The `/api/admin/audit-log` endpoint returned all records without pagination, causing timeouts on large datasets. Added cursor-based pagination with `limit` (default 50) and `before` cursor parameters.

### Frontend Changes
- Added KYC document viewer modal with image/pdf preview and download.
- Added System Config page with live preview of changes before saving.
- Added rollback history timeline on the System Config page.

---



## 2. AI Paid Shop Builder — Feature Completion

### What It Does
The AI Paid Shop Builder now supports merchant subscription plans and billing. Shop owners can upgrade their shop to a paid tier (Starter, Growth, Enterprise) with feature gating, usage limits, and automated invoicing. The AI agent was also enhanced to guide merchants through the upgrade flow conversationally.

### Subscription Plans

| Plan | Monthly Price (NGN) | Products Limit | Custom Domain | AI Assistant Messages/Month | Transaction Fee |
|---|---|---|---|---|---|
| Free | ₦0 | 10 | No | 50 | 2.5% |
| Starter | ₦5,000 | 100 | No | 500 | 2.0% |
| Growth | ₦15,000 | 1,000 | Yes | 5,000 | 1.5% |
| Enterprise | ₦50,000 | Unlimited | Yes | Unlimited | 1.0% |

### Endpoints Built

| Endpoint | Method | Description |
|---|---|---|
| `/api/user/shop/subscription` | `GET` | Get current shop subscription and usage stats |
| `/api/user/shop/subscription/plans` | `GET` | List all available plans with pricing |
| `/api/user/shop/subscription/upgrade` | `POST` | Upgrade shop to a paid plan |
| `/api/user/shop/subscription/cancel` | `POST` | Cancel subscription at end of billing period |
| `/api/user/shop/subscription/invoices` | `GET` | List past invoices (PDF download available) |
| `/api/user/shop/subscription/usage` | `GET` | Current month usage stats (products, AI messages, transactions) |
| `/api/webhooks/subscription` | `POST` | Receives payment provider callbacks for subscription renewals |

### AI Agent Subscription Flow
The AI agent in `/user/shop/ai/chat` now detects subscription-related intent and can:
- Explain plan differences and pricing
- Initiate the upgrade flow by calling `/api/user/shop/subscription/upgrade`
- Warn merchants when they are approaching plan limits (e.g., "You've used 8 of 10 product slots on the Free plan")

### Feature Gating
- Product creation endpoint returns `402 Payment Required` if the shop has exceeded its plan limit.
- Custom domain is only editable on Growth and Enterprise plans.
- AI chat returns a quota warning when the merchant is at 80% of their message limit.

### Database Tables Added

| Table | Purpose |
|---|---|
| `shop_subscriptions` | Active subscription per shop with plan, status, and billing cycle |
| `subscription_plans` | Plan definitions with limits, pricing, and feature flags |
| `subscription_invoices` | Generated invoices with PDF URL, payment status, and line items |
| `subscription_usages` | Daily usage snapshots for products, AI messages, and transactions |

### Billing & Invoicing
- Invoices are generated on the 1st of each month for active subscriptions.
- PDF invoices are stored in S3-compatible storage and emailed to the shop owner.
- Failed subscription payments trigger a 3-day grace period before the shop is downgraded to Free.

### Bug Fixes
- **AI agent not enforcing product limits:** The agent could suggest adding products beyond the plan limit, and the backend accepted them. Fixed by adding a pre-creation limit check in `ShopBuilderController`.
- **Subscription webhook race condition:** If a subscription renewal webhook arrived before the invoice was generated, the system marked the invoice as paid without a corresponding invoice record. Fixed by generating the invoice before sending the payment request to the provider.
- **Plan downgrade data loss:** Downgrading from a paid plan to Free did not truncate excess products or revert custom domains. Fixed by adding a downgrade handler that enforces Free plan limits and reverts domain settings.

---



## 3. Cron Jobs on Render — Setup & Monitoring

### What Was Done
The team set up scheduled background jobs on Render to handle periodic maintenance tasks that were previously run ad-hoc or not at all.

### Cron Jobs Configured

| Job | Schedule | Description |
|---|---|---|
| `payment-indexer` | Every 2 minutes | Polls CKB node and EVM chains for payment confirmations; updates `PaymentIntent` status |
| `webhook-dispatcher` | Every 5 minutes | Retries failed webhook deliveries with exponential backoff |
| `onramp-rate-sync` | Every 30 seconds | Fetches fresh exchange rates from liquidity providers |
| `recharge-bundle-sync` | Every 1 hour | Syncs data bundle catalog from recharge provider APIs |
| `subscription-invoice-generator` | Daily at 00:00 UTC | Generates monthly invoices for active subscriptions |
| `subscription-grace-period-check` | Daily at 06:00 UTC | Checks for subscriptions in grace period; downgrades expired ones |
| `stale-onramp-resolver` | Every 15 minutes | Auto-cancels onramp orders stuck in `processing` beyond timeout |
| `delivery-zone-sync` | Every 6 hours | Recalculates delivery zone data from partner APIs |

### Render Configuration
- All cron jobs are defined in `render.yaml` as separate background worker services.
- Each worker has its own `START_COMMAND` and `ENV` variables scoped to its function.
- Health check endpoints added to each worker for Render monitoring.

### Monitoring & Alerting
- Added structured logging to all cron jobs with job name, duration, and success/failure status.
- Set up Render alert policies:
  - Alert if a worker is down for > 2 minutes.
  - Alert if a job takes > 2x its normal duration (indicating external API slowdown).
  - Daily digest of failed webhook deliveries sent to the operations Slack channel.

### Bug Fixes
- **Cron job clock drift on Render free tier:** Render free tier workers can be paused, causing cron schedules to drift. Fixed by adding a "catch-up" check at job start: if the last run was missed, the job processes the missed window immediately.
- **Memory leak in payment indexer:** The 2-minute payment indexer was accumulating unclosed database connections. Fixed by explicitly closing the DB connection pool at the end of each run.
- **Webhook dispatcher duplicate retries:** If a worker restarted mid-batch, the next run would re-process already-delivered webhooks. Fixed by adding a `dispatched_at` timestamp check before retrying.

### Environment Variables Added

```env
# Cron / Workers
CRON_PAYMENT_INDEXER_INTERVAL_MS=120000
CRON_WEBHOOK_DISPATCHER_INTERVAL_MS=300000
CRON_ONRAMP_RATE_SYNC_INTERVAL_MS=30000
CRON_RECHARGE_BUNDLE_SYNC_INTERVAL_MS=3600000
CRON_SUBSCRIPTION_INVOICE_CRON=0 0 * * *
CRON_GRACE_PERIOD_CRON=0 6 * * *
CRON_STALE_ONRAMP_RESOLVER_INTERVAL_MS=900000

# Render
RENDER_SERVICE_TYPE=worker
RENDER_HEALTH_CHECK_PATH=/health
```

---



## 4. Files Modified / Created This Week

| # | File | Change |
|---|---|---|
| 1 | `app/Controllers/Http/Admin/MerchantController.ts` | Modified — added KYC endpoints |
| 2 | `app/Controllers/Http/Admin/ShopController.ts` | Added — force-publish/unpublish endpoints |
| 3 | `app/Controllers/Http/Admin/SystemConfigController.ts` | Added — platform config and rollback |
| 4 | `app/Controllers/Http/Admin/SubscriptionController.ts` | Added — admin subscription oversight |
| 5 | `app/Controllers/Http/User/ShopSubscriptionController.ts` | Added — merchant-facing subscription endpoints |
| 6 | `app/Controllers/Http/User/ShopSubscriptionWebhookController.ts` | Added — subscription payment webhooks |
| 7 | `app/Models/ShopSubscription.ts` | Added |
| 8 | `app/Models/SubscriptionPlan.ts` | Added |
| 9 | `app/Models/SubscriptionInvoice.ts` | Added |
| 10 | `app/Models/SubscriptionUsage.ts` | Added |
| 11 | `app/Models/PlatformSetting.ts` | Added |
| 12 | `app/Services/SubscriptionService.ts` | Added — plan enforcement, billing, downgrade |
| 13 | `app/Services/InvoiceService.ts` | Added — PDF generation and S3 storage |
| 14 | `app/Services/UsageTrackingService.ts` | Added — daily usage snapshot tracking |
| 15 | `app/Middleware/RequireRoleMiddleware.ts` | Modified — fixed RBAC bypass |
| 16 | `database/migrations/20260729000000_create_shop_subscriptions.ts` | Added |
| 17 | `database/migrations/20260729000001_create_subscription_plans.ts` | Added |
| 18 | `database/migrations/20260729000002_create_subscription_invoices.ts` | Added |
| 19 | `database/migrations/20260729000003_create_subscription_usages.ts` | Added |
| 20 | `database/migrations/20260729000004_create_platform_settings.ts` | Added |
| 21 | `render.yaml` | Modified — added 8 cron worker services |
| 22 | `app/Services/Cron/PaymentIndexerCron.ts` | Added |
| 23 | `app/Services/Cron/WebhookDispatcherCron.ts` | Added |
| 24 | `app/Services/Cron/OnrampRateSyncCron.ts` | Added |
| 25 | `app/Services/Cron/RechargeBundleSyncCron.ts` | Added |
| 26 | `app/Services/Cron/SubscriptionInvoiceCron.ts` | Added |
| 27 | `app/Services/Cron/GracePeriodCheckCron.ts` | Added |
| 28 | `app/Services/Cron/StaleOnrampResolverCron.ts` | Added |
| 29 | `app/Services/Cron/DeliveryZoneSyncCron.ts` | Added |
| 30 | `app/(admin)/admin/system-config/page.tsx` | Added |
| 31 | `components/admin/KycDocumentModal.tsx` | Added |
| 32 | `components/admin/RollbackTimeline.tsx` | Added |

---



## 5. Environment Variables Added

```env
# Subscription Billing
STRIPE_SECRET_KEY=...
STRIPE_WEBHOOK_SECRET=...
INVOICE_S3_BUCKET=...
INVOICE_S3_REGION=...

# Platform Config
PLATFORM_FEE_PERCENTAGE=2.5
MINIMUM_PAYOUT_AMOUNT=1000
MAX_ONRAMP_ORDER_NGN=500000
FEATURE_GUEST_CHECKOUT=true
FEATURE_ONRAMP=true

# Cron Workers
CRON_PAYMENT_INDEXER_INTERVAL_MS=120000
CRON_WEBHOOK_DISPATCHER_INTERVAL_MS=300000
CRON_ONRAMP_RATE_SYNC_INTERVAL_MS=30000
CRON_RECHARGE_BUNDLE_SYNC_INTERVAL_MS=3600000
CRON_SUBSCRIPTION_INVOICE_CRON=0 0 * * *
CRON_GRACE_PERIOD_CRON=0 6 * * *
CRON_STALE_ONRAMP_RESOLVER_INTERVAL_MS=900000
CRON_DELIVERY_ZONE_SYNC_INTERVAL_MS=21600000
```

---



## 6. Database Tables Added

| Table | Purpose |
|---|---|
| `shop_subscriptions` | Active subscription per shop with plan, status, and billing cycle |
| `subscription_plans` | Plan definitions with limits, pricing, and feature flags |
| `subscription_invoices` | Generated invoices with PDF URL, payment status, and line items |
| `subscription_usages` | Daily usage snapshots for products, AI messages, and transactions |
| `platform_settings` | Key-value platform configuration with version history |

---



## 7. Week 19 Metrics

| Area | Count |
|---|---|
| Admin endpoints added | 7 |
| Admin frontend pages added | 2 |
| Shop Builder endpoints added | 6 |
| New models created | 5 |
| New migrations added | 5 |
| Cron jobs configured on Render | 8 |
| Bugs fixed | 5 |
| RBAC roles enforced | 4 |

---



## 8. Key Lessons Learned
- Admin tooling is never "done" after the first build — operational feedback surfaces real gaps (KYC viewer, config rollback) that are invisible in internal testing.
- Feature gating must be enforced at the API layer, not just in the AI agent or frontend. The agent is a guide; the API is the gatekeeper.
- Cron jobs on Render free tier require explicit "catch-up" logic because workers can be paused without warning. Always design for missed runs.
- Webhook and dispatcher idempotency must account for worker restarts, not just duplicate messages from external providers.

---



## 9. Community Rollout Readiness

| Item | Status |
|---|---|
| Admin Section — KYC & Config | Ready for operations team |
| AI Paid Shop Builder | Ready for merchant TEST |
| Cron Jobs on Render | Deployed and monitored |
| Subscription Billing | Ready for TEST |
| Platform Config Panel | Ready for super_admin |
| Community Testing Window | Open |
