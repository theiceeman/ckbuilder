# Activity Log - Week 18

## Summary
Week 18 expanded the customer-facing checkout experience with **guest checkout capabilities**, **delivery settings management**, **promotional discount logic**, and **payment intent metadata enrichment**. The team also improved email notifications to include customers, hardened the payment indexer, and emitted cart lifecycle events via SSE. A frontend integration guide was added to support the new flows.

---



## 1. Guest Checkout & Cart

### What It Does
Customers can now purchase from shops without creating an account. Guest checkout is supported via public endpoints, while logged-in users retain their existing auth-based flow. Both paths calculate delivery fees, apply promo codes, and emit cart events in real time.

### Endpoints Built

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/cart/checkout` | `POST` | Public | Guest checkout — creates PaymentIntent with delivery + discount breakdown |
| `/api/cart/wallet` | `POST` | Public | Guest wallet-funded checkout |
| `/api/user/cart/checkout` | `POST` | Required | Logged-in checkout — same logic, linked to user account |
| `/api/user/cart/wallet` | `POST` | Required | Logged-in wallet-funded checkout |
| `/api/cart/items/:id` | `DELETE` | Public | Remove item from guest cart |
| `/api/cart/clear` | `POST` | Public | Clear guest cart |
| `/api/user/cart/items/:id` | `DELETE` | Required | Remove item from user cart |
| `/api/user/cart/clear` | `POST` | Required | Clear user cart |

### Checkout Flow
```
POST /api/cart/checkout
{
  "items": [{ "product_id": "prod_123", "quantity": 2 }],
  "promo_code": "SUMMER25",
  "delivery_address": { "state": "Lagos", "city": "Ikeja", "address": "15 XYZ Street" }
}
→ Delivery fee calculated from shop delivery settings
→ Promo code validated → discount applied (percentage or fixed)
→ PaymentIntent created with metadata:
  {
    "items_total": 30000,
    "delivery_fee": 2000,
    "discount_amount": 7500,
    "delivery_address": { ... },
    "customer_email": "guest@example.com"
  }
→ Customer redirected to payment page
```

### Cart Lifecycle Events (SSE)
The `SseService` now emits the following events for logged-in users:
- `cart.item_added`
- `cart.removed`
- `cart.updated`
- `cart.cleared`
- `cart.checkout_completed`

### Files Modified
- `app/Controllers/Http/CartController.ts` — Added public and auth-based checkout/wallet endpoints, cart clear and item removal.
- `routes/public.ts` — Registered new public cart routes.
- `routes/user/cart.ts` — Registered new auth-based cart routes.
- `routes/user/shop_builder.ts` — Updated to support new cart/checkout flows.

---



## 2. Delivery Settings

### What It Does
Shop owners can now configure per-shop delivery rules: flat fees, state-based delivery zones, free delivery toggle, and free delivery thresholds. Public endpoints allow the storefront to fetch and apply the correct delivery charges at checkout.

### Model & Migration
- **Model:** `ShopDeliverySetting`
- **Migration:** `20260723120000_create_shop_delivery_settings.ts`

### Fields
| Field | Type | Description |
|---|---|---|
| `shop_id` | UUID | Foreign key to shop |
| `delivery_fee` | Integer | Base flat delivery fee (NGN kobo) |
| `free_delivery` | Boolean | Toggle free delivery on/off |
| `free_delivery_threshold` | Integer | Minimum order value for free delivery (NGN kobo) |
| `delivery_zones` | JSONB | State-based fee overrides, e.g. `{ "Lagos": 1500, "Abuja": 2500 }` |
| `is_active` | Boolean | Enable/disable delivery for the shop |

### Endpoints

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/user/shop/delivery-settings` | `GET/PUT` | Required | Shop owner fetches and updates delivery settings |
| `/api/shop/:subdomain/delivery-settings` | `GET` | Public | Storefront fetches active delivery settings for checkout |

### Checkout Integration
At checkout, the system:
1. Fetches the shop's active `ShopDeliverySetting`.
2. Checks if the order total meets `free_delivery_threshold` → if so, fee = 0.
3. Looks up the customer's state in `delivery_zones` → applies zone-specific fee if present.
4. Falls back to `delivery_fee` if no zone match.

### Files Added / Modified
- `app/Controllers/Http/ShopDeliverySettingController.ts` — New controller for delivery settings CRUD.
- `app/Models/ShopDeliverySetting.ts` — New model.
- `database/migrations/20260723120000_create_shop_delivery_settings.ts` — New migration.

---



## 3. Payment Intent Enhancements

### What It Does
`PaymentIntent` was extended to support guest checkout, richer checkout context, and customer linking by email.

### Migrations
| Migration | Change |
|---|---|
| `20260722150500_add_customer_to_payment_intents.ts` | Added `customer_id` (nullable UUID) and `customer_email` (string) |
| `20260723120001_add_metadata_to_payment_intents.ts` | Added `metadata` JSONB field |

### New Fields
| Field | Type | Description |
|---|---|---|
| `customer_id` | UUID (nullable) | Links to `users.id` for logged-in customers |
| `customer_email` | String | Stores email for guest customers (used for notifications and lookup) |
| `metadata` | JSONB | Stores checkout breakdown: `items_total`, `delivery_fee`, `discount_amount`, `delivery_address`, `promo_code`, etc. |

### Impact
- Guest payments are linked to a customer record by email, enabling order history and support workflows without requiring account creation.
- `metadata` gives the frontend and support team full visibility into how a payment was calculated, simplifying reconciliation and dispute resolution.

### Files Modified
- `app/Models/PaymentIntent.ts` — Updated model with new fields.

---



## 4. Email Notifications

### What It Does
Email notifications were extended to include the customer (previously only the business was notified). Customers now receive an order confirmation email when a payment is confirmed.

### Changes
- **New template:** `app/Lib/notification/email-templates/customer_order_confirmed.html` — customer-facing order confirmation with items, amounts, and delivery details.
- **Updated service:** `app/Services/EmailNotificationService.ts` — Now sends to both `business.email` and `customer_email` (from `PaymentIntent`) when an order is confirmed.

### Email Types in Place
| Template | Recipient | Trigger |
|---|---|---|
| `payment_confirmed.html` | Business | Payment confirmed on-chain |
| `payment_failed.html` | Business | Payment failed or expired |
| `customer_order_confirmed.html` | Customer | Order confirmed and payment received |

### Files Added / Modified
- `app/Lib/notification/email-templates/customer_order_confirmed.html` — New template.
- `app/Services/EmailNotificationService.ts` — Updated to include customer notification.

---



## 5. Payment Indexer

### What It Does
The `PaymentIndexerService` was enhanced to more reliably track payment status transitions and improve resilience when dispatching webhooks and SSE events.

### Enhancements
- Improved status tracking logic to handle edge cases where on-chain confirmation and webhook delivery are delayed or out of order.
- Added resilience in the webhook/dispatcher path: transient failures no longer block status updates; events are retried with backoff.
- Better idempotency handling to prevent duplicate webhook deliveries for the same payment state change.

### Files Modified
- `app/Services/PaymentIndexerService.ts` — Enhanced status tracking and dispatcher resilience.

---



## 6. Shop Builder

### What It Does
Shop builder routes were updated to support the new guest checkout and cart flows while maintaining backward compatibility with existing endpoints.

### Changes
- `routes/user/shop_builder.ts` now proxies the updated cart/checkout endpoints.
- Existing shop builder endpoints (products, themes, AI chat) remain unchanged.

### Files Modified
- `routes/user/shop_builder.ts` — Updated routing for new cart/checkout flows.

---



## 7. Documentation

### What It Does
A frontend integration guide was added to help developers and shop owners implement the new guest checkout, delivery settings, promo codes, and cart flows on the frontend.

### File Added
- `docs/frontend-changes-guest-checkout-delivery-promos.md` — Full guide covering:
  - Guest cart with localStorage
  - Checkout flow for guest and logged-in users
  - Delivery fee calculation logic
  - Promo code application
  - SSE event subscription for cart updates

---



## 8. Database Tables Added

| Table | Purpose |
|---|---|
| `shop_delivery_settings` | Per-shop delivery fees, zones, free delivery rules |
| (Modified) `payment_intents` | Added `customer_id`, `customer_email`, `metadata` fields |

---



## 9. Environment Variables Added

```env
# Email Notifications
RESEND_API_KEY=...
FROM_EMAIL=noreply@wtpayments.com
CUSTOMER_NOTIFICATION_ENABLED=true
```

---



## 10. Files Modified / Created This Week

| # | File | Change |
|---|---|---|
| 1 | `app/Controllers/Http/CartController.ts` | Modified — added public and auth checkout/wallet endpoints, cart clear/remove |
| 2 | `app/Controllers/Http/ShopDeliverySettingController.ts` | Added — delivery settings CRUD |
| 3 | `app/Models/ShopDeliverySetting.ts` | Added |
| 4 | `app/Models/PaymentIntent.ts` | Modified — added customer_id, customer_email, metadata |
| 5 | `app/Services/EmailNotificationService.ts` | Modified — added customer order confirmation |
| 6 | `app/Services/PaymentIndexerService.ts` | Modified — enhanced status tracking and dispatcher resilience |
| 7 | `app/Lib/notification/email-templates/customer_order_confirmed.html` | Added |
| 8 | `database/migrations/20260723120000_create_shop_delivery_settings.ts` | Added |
| 9 | `database/migrations/20260723120001_add_metadata_to_payment_intents.ts` | Added |
| 10 | `routes/public.ts` | Modified — registered public cart routes |
| 11 | `routes/user/cart.ts` | Modified — registered auth cart routes |
| 12 | `routes/user/shop_builder.ts` | Modified — updated for new cart/checkout flows |
| 13 | `docs/frontend-changes-guest-checkout-delivery-promos.md` | Added |

---



## 11. Pending / Next Steps

| Item | Owner | Priority |
|---|---|---|
| Fiber node `EHOSTUNREACH` retry/backoff logic or failover node URL | Backend | High |
| Frontend implementation of guest cart (localStorage) and checkout flows per new docs | Frontend | High |
| Payment webhook reconciliation testing end-to-end | Backend + QA | High |

---



## 12. Week 18 Metrics

| Area | Count |
|---|---|
| New public endpoints | 4 |
| New auth endpoints | 4 |
| New models | 1 (`ShopDeliverySetting`) |
| New migrations | 2 |
| New email templates | 1 |
| SSE event types added | 5 |
| Bugs fixed / enhancements | 2 (indexer resilience, delivery integration) |

---



## 13. Community Rollout Readiness

| Item | Status |
|---|---|
| Guest Checkout | Ready for TEST env |
| Delivery Settings | Ready for TEST env |
| Promo Codes / Discounts | Ready for TEST env |
| Customer Email Notifications | Ready |
| Payment Indexer Resilience | Improved; monitoring in place |
| Frontend Guest Cart | Pending implementation per docs |
