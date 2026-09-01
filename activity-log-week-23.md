# Activity Log - Week 23

## Summary

Week 23 focused on **Multi-Network Crypto Integration** — extending the platform to support Solana, Tron, EVM, and CKB networks across backend payloads, frontend types, wallet dashboards, withdrawal flows, transaction history, guest checkout, and storefront functionality. This work expands CKB's utility beyond a single chain, positioning WT Payments as a true multi-network crypto payment gateway.

---

## 1. Backend Payloads & Response Integration

### What It Does
Reviewed and integrated backend contract changes for Solana and Tron networks across all relevant endpoints, ensuring the frontend correctly handles new payload shapes.

### Types Added
- `NetworkType = "evm" | "ckb" | "solana" | "tron"`
- `AvailableAsset`, `AvailableAssetCrypto`, `AvailableAssetNetwork` — supporting `networkType`, `chainKey`, `chainId`, `contractAddress`, and `isTestnet`
- `WithdrawalHistoryItem` and `WithdrawalSSEEventData` — matching new backend shapes

### Endpoints Covered
- Available assets
- Withdrawal initiate/confirm
- Payment intent history
- Withdrawal history
- SSE events
- Wallet creation

---

## 2. Available Assets

### What Changed
Updated asset display components to read contract/mint addresses from the new backend shape.

### Files Updated
| File | Change |
|------|--------|
| `app/(dashboard)/dashboard/currency/page.tsx` | Updated `AssetItem` type for new network fields |
| `src/components/currency/CurrencyTable.tsx` | Reads `asset.crypto.contractAddress || asset.network?.contract_address` |
| `src/components/currency/CurrencyMobile.tsx` | Reads `asset.crypto.contractAddress || asset.network?.contract_address` |
| `src/components/overview/AvailableAssetCard.tsx` | Updated `AssetItem` type for new network fields |

---

## 3. Wallet Dashboard / Balances

### What Changed
Replaced single aggregated balance with per-wallet cards showing network-specific information.

### New Features
- Each wallet card shows: network logo, network name, balance, currency symbol, truncated address, and testnet badge
- Added `UserWallet` type with `cryptoNetwork` and `currency` preloaded objects
- Added `WalletsResponse` type for `GET /api/user/wallets`

### File Updated
- `src/components/wallet/WalletSummaryCards.tsx` — per-network wallet cards with logo, balance, truncated address, testnet badge

---

## 4. Withdrawal Flow — Wallet Selector

### What Changed
Added a wallet selector dropdown in the withdrawal page, enabling users to choose which wallet to withdraw from.

### New Features
- Fetches `GET /api/user/wallets` on mount and filters by active status
- Each wallet option shows network logo, balance, truncated address, and active/inactive badge
- Auto-populates `user_wallet_id`, `network_id`, and `crypto_currency_id` from selected wallet
- Network-aware address validation:
  - EVM: `0x...`
  - Solana: Base58
  - Tron: `T...`
  - CKB: `ckt1q...`
- Submit button and amount balance reflect selected wallet's currency symbol

### File Updated
- `app/(dashboard)/dashboard/wallet/withdraw/page.tsx` — wallet selector dropdown, network-aware address validation, SSE completion toast

---

## 5. Transaction History

### What Changed
Updated transaction history types and components to display new fields: `crypto_amount`, `crypto_currency`, `tx_hash`, `wallet_address`, `user_wallet_id`.

### Files Updated
| File | Change |
|------|--------|
| `lib/payment-intent-history.ts` | Added new fields: `crypto_amount`, `crypto_currency`, `tx_hash`, `wallet_address`, `user_wallet_id` |
| `src/components/wallet/WithdrawalHistoryTable.tsx` | Uses new `WithdrawalHistoryItem` shape (`method`, `crypto_currency`, `wallet`, `network`, `tx_hash`) |
| `src/components/transactions/TransactionsTable.tsx` | Displays `txHash` under amount when present |
| `src/components/overview/TransactionsTable.tsx` | Displays `txHash`, network badge |

### Dynamic Icons Added
SOL, TRX, USDC, USDT, CKB, ETH, MATIC

---

## 6. SSE Events — Wallet Balance Updates

### What Changed
Updated wallet balance hook to merge incoming wallet updates with previous state instead of replacing, enabling granular per-wallet balance updates without losing other wallets.

### File Updated
- `hooks/use-wallet-balance.ts` — `mergeWallets()` for granular updates; retained `withdrawal.updated` listener for real-time completion notifications

---

## 7. Payout Settings — Crypto Wallet

### What Changed
Added a crypto wallet option to payout settings, allowing users to receive payouts to a crypto address instead of a bank account.

### New Features
- Bank Account / Crypto Wallet toggle
- Crypto payout form accepting EVM, Solana, Tron, and CKB addresses
- Network-aware validation with inline error messages and placeholder hints
- Posts `type: "CRYPTO"` with `wallet_address`, `network_type`, `currency_id`

### File Updated
- `src/components/settings/PayoutSettingsSection.tsx` — crypto wallet form with network-aware validation

---

## 8. Transaction Creation — Blockchain & Currency Selectors

### What Changed
Updated the transaction creation page to fetch available assets and present blockchain/currency selectors.

### New Features
- Fetches `/backend/available-assets` on mount
- Blockchain selector using `SelectBlockchainSheet`, populated from available assets
- Cryptocurrency selector using `SelectAssetSheet`, filtered by chosen blockchain
- Both selectors render network/asset logos from API with text fallbacks
- Passes real `crypto_currency_id` to `POST /user/payment-intent/create-wallet`
- Added validation requiring both blockchain and currency before submission

### File Updated
- `app/transactions/create/page.tsx` — blockchain/currency selectors, validation, logo/networkType propagation

---

## 9. WaitingForPaymentModal Enhancements

### What Changed
Enhanced the payment modal to show network-specific information and estimated arrival times.

### New Features
- Extended `PaymentIntentData.crypto` with optional `logo` and `networkType`
- Added `renderNetworkBadge()` with network-specific colors:
  - Solana: purple
  - Tron: red
  - EVM: blue
  - CKB: green
- Shows asset logo next to amount and colored network badge
- Added `Estimated arrival` row based on network type
- Updated footer text to reflect actual selected network instead of hardcoded "Fiber protocol"

### File Updated
- `components/WaitingForPaymentModal.tsx` — network badge, asset logo, estimated arrival row

---

## 10. Storefront & Shop Fixes

### What Changed
Fixed multiple issues with the shop builder and storefront functionality.

### Fixes
| Issue | Fix |
|-------|-----|
| `/dashboard/shop` "Create Your Shop" button non-functionality | Moved `CreateShopModal` into empty-state block in `app/(dashboard)/dashboard/shop/page.tsx` |
| Backend `E_ROUTE_NOT_FOUND: Cannot POST:/user/shop` | Updated Next.js rewrite in `next.config.ts` to include `/api/` prefix |
| Double `/api/api/` path in storefront | Switched to `/storefront/:subdomain` backend endpoints in `app/shop/[subdomain]/page.tsx` |
| Redundant product fetch | Removed redundant fetch since storefront controller embeds products |
| Double `/api/` path bugs | Fixed in `ApiKeysSection.tsx`, `WebhooksSection.tsx`, and `WaitingForPaymentModal.tsx` SSE URL |
| "Add to Cart" not showing | Removed `product.is_active` gating in `app/shop/[subdomain]/page.tsx` |
| "View Cart" redirect for unauthenticated | Redirects to `/login` instead of `/cart` |

### Files Updated
| File | Change |
|------|--------|
| `app/(dashboard)/dashboard/shop/page.tsx` | Moved CreateShopModal into empty-state block |
| `next.config.ts` | Added `/api/` prefix to rewrite |
| `app/shop/[subdomain]/page.tsx` | Switched to `/storefront/:subdomain`, removed redundant product fetch |
| `src/components/settings/ApiKeysSection.tsx` | Fixed double `/api/` path |
| `src/components/settings/WebhooksSection.tsx` | Fixed double `/api/` path |
| `components/WaitingForPaymentModal.tsx` | Fixed SSE URL double `/api/` path |
| `app/api/user/prices/route.ts` | Created proxy for live price polling |
| `lib/usePrices.ts` | Created hook for 60-second live price polling |

---

## 11. Guest Checkout

### What Changed
Made the storefront cart fully usable without authentication, enabling guest checkout.

### New Features
- `addToCart`, `updateCartItem`, `removeCartItem` now use localStorage guest cart when no token is present
- Storefront "View Cart" button wired to public `/checkout` page instead of `/dashboard/cart`
- Created guest checkout page at `app/checkout/page.tsx` with cart review, guest details form, and crypto checkout flow
- Guest checkout calls public backend endpoints without auth: `/backend/cart/checkout` and `/backend/cart/wallet`
- Created Next.js proxy routes:
  - `app/api/cart/checkout/route.ts` — forwards to backend public guest endpoints
  - `app/api/cart/wallet/route.ts` — forwards to backend public guest endpoints
- Guest checkout sends `customer_email`, `items` with `product_id`, `quantity`, `price`, `shopId`
- Wallet request uses `reference_id` per new backend contract
- Added success redirect and cart cleanup on payment completion
- Storefront guest cart stores `shop_id` on each item so checkout includes `shopId` in items payload

### Checkout Page Enhancements
- `/checkout` loads cart from server API when user is logged in, falling back to localStorage guest cart when not
- Added delivery settings fetch via `/backend/shop/[subdomain]/delivery-settings`
- Added delivery address form fields, state selector, promo code input, and order summary
- Order summary shows `items_total`, `delivery_fee`, `discount_amount`, and backend `fiat_amount`
- Updated checkout payload: `customer_email`, `items` with `product_id`, `quantity`, `price`, `shopId`, `delivery_address`, `delivery_state`, optional `promo_code`
- Created `/checkout/success` page with reference ID display
- Created `app/api/shop/[subdomain]/delivery-settings/route.ts` proxy for public delivery settings
- Fixed `/checkout` validation bug: `canProceed` now checks correct `delivery` fields after syncing `guestName` and `guestState`

### Files Created / Updated
| File | Change |
|------|--------|
| `app/checkout/page.tsx` | Created — guest checkout page |
| `app/checkout/success/page.tsx` | Created — success page with reference ID |
| `app/api/cart/checkout/route.ts` | Created — proxy for public guest checkout |
| `app/api/cart/wallet/route.ts` | Created — proxy for public guest wallet creation |
| `app/api/shop/[subdomain]/delivery-settings/route.ts` | Created — proxy for public delivery settings |
| `app/shop/[subdomain]/page.tsx` | Updated cart to use localStorage for guests |

---

## 12. Product Management

### What Changed
Added image upload functionality to the product creation/editing form.

### New Features
- Image upload UI in `app/(dashboard)/dashboard/shop/products/page.tsx`
- Pending image previews with remove buttons
- Images upload after product save

### File Updated
- `app/(dashboard)/dashboard/shop/products/page.tsx` — image upload UI with previews

---

## 13. Cart / Crypto Checkout

### What Changed
Updated cart checkout to use the new backend endpoint for wallet creation during crypto payment.

### New Features
- Added `payment_intent_id` to `CheckoutResult` type
- Updated `handleCreateWallet` to call `/api/user/cart/wallet` (POST `{ payment_intent_id, crypto_currency_id }`) instead of old endpoint
- Created Next.js proxy route at `app/api/user/cart/wallet/route.ts`
- Preserved wallet modal SSE flow and crypto/fiat data mapping

### Files Updated
| File | Change |
|------|--------|
| `app/(dashboard)/dashboard/cart/page.tsx` | Updated wallet creation to new endpoint |
| `app/api/user/cart/wallet/route.ts` | Created — proxy for cart wallet creation |

---

## 14. How This Helps CKB

### Multi-Network Reach
By integrating Solana, Tron, EVM, and CKB, WT Payments becomes a **universal crypto payment gateway** rather than a single-chain solution. Merchants can accept payments in any of these networks, dramatically expanding the potential user base.

### Merchant Flexibility
- Merchants choose their preferred network based on fee structure, speed, and customer base
- Withdrawal to any supported network increases liquidity options
- Multi-network support future-proofs the platform against chain-specific volatility

### Customer Experience
- Customers pay with their preferred crypto without needing to bridge or swap
- Network-specific validation prevents lost funds from incorrect address formats
- Real-time SSE updates provide consistent UX across all networks

### Technical Foundation
- Type-safe network abstraction (`NetworkType`) makes adding future networks straightforward
- Per-wallet balance cards give users clear visibility into their multi-chain holdings
- Network-aware address validation reduces user error and support tickets

### Storefront Growth
- Guest checkout removes friction for first-time buyers
- Delivery settings integration enables physical product sales
- Image upload for products improves storefront quality
- Fixes to "Add to Cart" and checkout flow ensure a working purchase path

---

## 15. Files Modified / Created

| # | File | Change |
|---|---|---|
| 1 | `types/index.ts` | Added `NetworkType`, `AvailableAsset*`, `UserWallet`, `WithdrawalSSEEventData`, `WithdrawalHistoryItem` |
| 2 | `src/components/wallet/WalletSummaryCards.tsx` | Per-network wallet cards |
| 3 | `hooks/use-wallet-balance.ts` | Granular wallet merging, SSE support |
| 4 | `app/(dashboard)/dashboard/wallet/withdraw/page.tsx` | Wallet selector, network-aware validation |
| 5 | `src/components/wallet/WithdrawalHistoryTable.tsx` | New API shape, dynamic crypto icons |
| 6 | `lib/payment-intent-history.ts` | New fields for multi-network history |
| 7 | `src/components/transactions/TransactionsTable.tsx` | `txHash` display |
| 8 | `src/components/overview/TransactionsTable.tsx` | `txHash` display, network badge |
| 9 | `app/(dashboard)/dashboard/currency/page.tsx` | Updated `AssetItem` type |
| 10 | `src/components/currency/CurrencyTable.tsx` | Contract/mint address read |
| 11 | `src/components/currency/CurrencyMobile.tsx` | Contract/mint address read |
| 12 | `src/components/overview/AvailableAssetCard.tsx` | Updated type |
| 13 | `src/components/settings/PayoutSettingsSection.tsx` | Crypto wallet form |
| 14 | `app/transactions/create/page.tsx` | Blockchain/currency selectors |
| 15 | `components/WaitingForPaymentModal.tsx` | Network badge, asset logo, estimated arrival |
| 16 | `app/(dashboard)/dashboard/shop/page.tsx` | CreateShopModal fix |
| 17 | `next.config.ts` | Added `/api/` prefix rewrite |
| 18 | `app/shop/[subdomain]/page.tsx` | Storefront fixes, removed redundant fetch |
| 19 | `src/components/settings/ApiKeysSection.tsx` | Fixed double `/api/` path |
| 20 | `src/components/settings/WebhooksSection.tsx` | Fixed double `/api/` path |
| 21 | `app/api/user/prices/route.ts` | Created — live price polling proxy |
| 22 | `lib/usePrices.ts` | Created — 60-second price polling hook |
| 23 | `app/checkout/page.tsx` | Created — guest checkout page |
| 24 | `app/checkout/success/page.tsx` | Created — success page |
| 25 | `app/api/cart/checkout/route.ts` | Created — guest checkout proxy |
| 26 | `app/api/cart/wallet/route.ts` | Created — guest wallet creation proxy |
| 27 | `app/api/shop/[subdomain]/delivery-settings/route.ts` | Created — delivery settings proxy |
| 28 | `app/(dashboard)/dashboard/shop/products/page.tsx` | Image upload UI |
| 29 | `app/(dashboard)/dashboard/cart/page.tsx` | Updated wallet creation endpoint |
| 30 | `app/api/user/cart/wallet/route.ts` | Created — cart wallet proxy |

---

## 16. Verification

- **TypeScript:** All new types compile without errors
- **Dev server:** Checkout, wallet, withdrawal, and shop pages load without runtime errors
- **Proxy routes:** All new API proxy routes return 200 for happy-path requests
- **Guest checkout:** Full flow from cart → checkout → payment works without authentication
- **Network validation:** Address validation correctly rejects invalid formats for each network

---

## 17. Week 23 Metrics

| Area | Count |
|------|-------|
| Networks added | 3 (Solana, Tron, CKB on top of EVM) |
| New TypeScript types | 6+ |
| Pages created | 3 (`/checkout`, `/checkout/success`, proxy routes) |
| Components updated | 12+ |
| Storefront fixes | 7 |
| Guest checkout features | 5 |
| New proxy routes | 5 |

---

## 18. Key Lessons Learned

- **Network abstraction is critical** — a single `NetworkType` enum and consistent `networkType` field across all payloads prevents subtle bugs when switching between chains
- **Guest checkout is a conversion multiplier** — requiring authentication before purchase drops conversion significantly; localStorage guest carts provide a seamless experience
- **Proxy routes as adapters** — Next.js API routes act as translation layers between frontend expectations and backend contracts, making backend changes less disruptive
- **Address validation prevents user error** — network-specific regex patterns catch mistakes before they become on-chain losses
- **SSE over polling for real-time data** — wallet balance updates and withdrawal status are smoother and more reliable with Server-Sent Events

---

## 19. Next Steps

| Priority | Task |
|----------|------|
| High | Add Solana and Tron wallet creation endpoints if not already present in backend |
| High | Test actual on-chain transactions for each network in staging |
| High | Add network fee estimation to checkout and withdrawal flows |
| Medium | Implement network-specific error handling for failed transactions |
| Medium | Add multi-network portfolio view aggregating balances across all wallets |
| Low | Add network switching in shop builder for merchants who want to choose settlement chain |
