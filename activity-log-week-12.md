# Activity Log - Week 12

## Summary
Week 12 was the most front-end intensive week to date. We completed the full integration of the **wt-payment-dashboard** (Next.js 14 · TypeScript · Tailwind CSS) with the `wt-payments-server` backend. Every major feature was wired to a live endpoint — no mock data remains in production flows. The app is now a working crypto payment acceptance dashboard for Nigerian merchants dealing with real-time balances, withdrawals, and transaction analytics.

**Frontend URL:** https://wt-payments-frontend-aussvur3q-franklin-s-projects.vercel.app/

---

## Features Built & Integrated This Week

### 1. Environment Toggle (Live / Test Mode)
- Built header toggle that switches the account between **LIVE** and **TEST** modes.
- Persists state via `localStorage`; calls `POST /user/settings/general/switch-environment` with a Bearer token.
- Toggle is disabled while the request is in flight (prevents double-submit).
- Toast error surfaces if the account is not yet verified and cannot switch to LIVE.

### 2. Analytical Transaction Chart (Live Data)
- Replaced all static/hardcoded bar chart data with live API data from `GET /dashboard/analytical-transactions?period=week|month`.
- Added Week / Month toggle; bar with highest count gets the purple gradient highlight.
- Tooltip shows label, count, and USD amount on hover.
- Loading state uses `SectionLoader` (bars variant); errors display inline.

### 3. Real-Time Wallet Balance via SSE
- Created a shared `useWalletBalance()` React hook opening a persistent SSE connection to `GET /user/stream`.
- Listens for `wallet.balance_updated` events and updates React state instantly — no polling.
- Both the Wallet page and Dashboard "Total Wallet Balance" card update in real-time when a payment confirms.
- Token passed as `?token=` query param (native `EventSource` does not support custom headers).

### 4. Loading Animator Suite (Branded UI)
Built a complete set of branded loading states:

| Component | Used for |
|---|---|
| `PageLoader` | Full-screen splash with orbit rings, W logo, progress bar, % counter |
| `RouteLoaderBar` | Thin violet gradient progress bar at top of page during navigation |
| `SkeletonCard` | Shimmer placeholders (dashboard, transaction, stats, profile variants) |
| `ButtonSpinner` | Inline spinner inside buttons during async actions |
| `SectionLoader` | Mid-card loader (orbit, dots, bars, ring variants) |
| `OverlayLoader` | Frosted-glass overlay for modals/drawers while data loads |

### 5. Page-Level Loading (Next.js App Router)
- Created `loading.tsx` files at every route segment (root, dashboard, transactions).
- Next.js automatically shows `PageLoader` during page transitions — no custom logic needed.

### 6. Route Progress Bar
- `RouteLoaderBar` injected once into the root layout; shows on every client-side navigation.
- Detects route changes by polling `window.location.pathname` and animates accordingly.

### 7. Withdrawal Flow (3-Step: Quote → Initiate → Confirm)
Complete end-to-end withdrawal:

- **Step 1 — Live Fee Quote:** 300ms debounced call to `GET /user/withdrawal/quote` on each amount keypress; fee card updates live showing amount, fees, network fee, arrival time, and NGN exchange rate.
- **Step 2 — Initiate:** Calls `POST /user/withdrawal/initiate`; stores returned `otp_id`.
- **Step 3 — OTP Confirm:** 6-digit OTP modal calls `POST /user/withdrawal/confirm`; navigates to wallet on success.
- Crypto mode: network selector, asset selector, recipient wallet address.
- Fiat mode: bank name, account number, bank code, account holder name; NGN conversion in fee summary.
- All backend error messages (insufficient balance, invalid OTP, expired OTP) surface as toast notifications.

### 8. API Integration Audit
Audited the full codebase and replaced all internal Next.js proxy route calls (`/api/...`) with direct calls to `${NEXT_PUBLIC_API_BASE_URL}`:

| File | Endpoint |
|---|---|
| `lib/payment-intent-history.ts` | `/user/payment-intent/history` |
| `WithdrawalHistoryTable.tsx` | `/user/withdrawals/history` |
| `WithdrawalsCard.tsx` | `/user/withdrawals/history` |
| `PayoutPieChart.tsx` | `/dashboard/payout-chart` |
| `AnalyticalTransactionChart.tsx` | `/dashboard/analytical-transactions` |
| `use-wallet-balance.ts` | `/user/stream` (SSE) |

All requests include `Authorization: Bearer <token>` from `localStorage`.

### 9. Section Loaders Wired Across App
Replaced all plain `"Loading..."` strings with typed `SectionLoader` variants:

| Component | Variant |
|---|---|
| `AnalyticalTransactionChart` | `bars` |
| `TransactionsTable` | `bars` |
| `TransactionsMobile` | `bars` |
| `AvailableAssetCard` | `orbit` |
| `PayoutPieChart` | `ring` |

---

## Technical Stack
- **Framework:** Next.js 14 (App Router)
- **Language:** TypeScript
- **Styling:** Tailwind CSS
- **Charts:** Recharts
- **Real-time:** Server-Sent Events (native `EventSource`)
- **Auth:** JWT via `localStorage` + `Authorization: Bearer`
- **State:** React hooks (`useState`, `useEffect`, `useRef`)

---

## What Is Missing: CKB Integration

The current system works entirely with the backend's internal accounting layer and a standard fiat/crypto withdrawal flow. CKB is not yet integrated — the following are the concrete places where CKB features can and should be added:

### Where CKB Plugs In (Feature-by-Feature)

| Current Feature | CKB Integration Opportunity |
|---|---|
| Wallet balance via SSE | Replace or augment backend balance tracking with live CKB indexer cell queries; emit SSE event from indexer tip instead of internal DB |
| Withdrawal flow (crypto mode) | Wire `POST /user/withdrawal/initiate` to build and broadcast a real CKB transaction via Lumos/CCC SDK rather than a mock transfer |
| Analytical chart | Add CKB on-chain transaction history as a separate data series alongside internal records |
| Settlement currency picker | Add CKB and xUDT stablecoins as settlement options alongside the existing fiat/crypto options |
| Environment toggle (Live/Test) | Map LIVE mode to CKB mainnet and TEST mode to CKB testnet; pass the correct RPC endpoint to the backend |
| Payment confirmation flow | Replace internal "confirmed" status with on-chain confirmation: payment is confirmed only when the CKB transaction is included in a block and the indexer tip advances past it |
| Fee quote (Step 1 of withdrawal) | Include CKB on-chain fee estimation (cycles × fee rate) in the quote breakdown card |

### Immediate CKB Next Steps
1. **Wire the backend withdrawal endpoint to a real CKB transaction builder** — use Lumos or CCC SDK to construct, sign, and broadcast the transaction. The frontend already sends all the required fields; only the backend handler changes.
2. **Swap internal balance tracking to CKB indexer** — query live cells on the CKB indexer for the merchant's address; push balance changes to the existing SSE endpoint so the frontend `useWalletBalance()` hook picks them up automatically.
3. **Add xUDT stablecoins to the withdrawal and settlement picker** — the UI already supports multiple asset types; only the backend and the `AssetSelector` sheet need to know about the xUDT type hash.
4. **Map TEST/LIVE toggle to CKB testnet/mainnet** — the environment toggle is already built and wired; the backend just needs to read the environment flag and point to the correct CKB RPC endpoint.
5. **On-chain payment confirmation** — update the payment status polling logic to check CKB block inclusion before marking a payment as "confirmed" and updating the dashboard chart.

---

## Nigeria / Inflation Context
The withdrawal's NGN conversion display already shows merchants the real-time value of their crypto balance in local currency. Once CKB and xUDT stablecoins are wired:
- Merchants can receive payments in a stablecoin xUDT, protecting value against NGN inflation.
- They can withdraw to fiat when they need it, or hold on-chain to preserve purchasing power.
- The fee quote card will show transparent, predictable CKB on-chain fees rather than opaque internal fees.

---

## Week 12 Takeaway
Full frontend-to-backend integration is complete. The app is functionally working with the internal accounting layer. The next engineering sprint should focus entirely on replacing the internal layer with real CKB on-chain transactions — starting with the withdrawal flow and balance tracking, which are the two features the frontend already surfaces most prominently to merchants.
