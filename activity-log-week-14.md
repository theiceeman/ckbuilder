# Activity Log - Week 14

## Summary
Week 14 focused on stabilization and hardening of two major platform features introduced in Week 13: **API Key Generation** and the **AI Shop Builder**. Both features were completed but contained edge-case bugs and environment-specific issues that surfaced during initial testing. This week resolved all known bugs, validated cross-environment behavior (TEST vs LIVE), and began backend hosting preparations for community rollout.




## 1. API Key Generation — Debug & Fixes

### What Was Wrong
- **Environment isolation broken:** The `environment` filter was not being applied consistently when querying or generating keys. Both TEST and LIVE key pairs were sometimes returned or generated for the wrong context.
- **Raw secret key exposure risk:** In certain code paths (e.g., verification endpoint and webhook log viewer), the full `secret_key_hash` lookup was bypassed and the raw key material could surface in logs or API responses.
- **Verify endpoint mismatch:** The `POST /api/user/settings/api-key/verify` endpoint was comparing against the hashed value using a plain string comparison instead of a constant-time hash comparison, creating a timing oracle.
- **Environment toggle lag:** Switching `current_environment` with `PUT /api/user/settings/general` did not clear in-memory key caches, causing the wrong active key pair to be used for subsequent webhook signing and payment calls.

### Fixes Applied
- Introduced explicit environment: 'TEST' | 'LIVE'` check in every key-related DB query.
- Ensured `secret_key_hash` is computed at ingestion time and all lookups use `bcrypt/crypto` comparison — raw key is never logged or returned after initial creation.
- Replaced plain `===` comparison in `verifySecretKey` with `crypto.timingSafeEqual` style buffer comparison.
- Added cache invalidation hook inside the environment-switch mutation so active secret/public key references update immediately.

### Status
**Stable.** Live and TEST key generation, retrieval, and verification are now isolated and consistent.

---



## 2. AI Shop Builder — Debug & Completion

### What Was Wrong
- **Subdomain collision:** Shop creation did not enforce uniqueness at the database constraint level. Two merchants could register the same subdomain, causing routing conflicts.
- **Theme deserialization failure:** The AI agent returns `theme_config` as a JSON object, but prior toptest parsing left some values as strings (e.g. `"null"` instead of `null`). The frontend theme renderer crashed on non-recoverable types.
- **Memory context bleed:** Because `ai_shop_conversations` was queried without an `environment` or `shopId` boundary in one code path, agents could read memory from shops they did not own.
- **Product image upload gap:** The multipart endpoint accepted images but never purged orphaned image records when a product was updated, leading to stale image references and 404s in the shop frontend.
- **Publish transition not atomic:** `PUT /user/shop` with `{ "status": "published" }` did not validate that the shop had at least one product before flipping to published, resulting in empty storefronts surfacing to customers.

### Fixes Applied
- Added a unique index on `shops.subdomain` and returned a clear `409 Conflict` if the subdomain is taken.
- Added a JSON schema validator step before saving `theme_config` from AI responses; invalid schema now triggers a fallback to the system default theme instead of a crash.
- Tightened all conversation queries to require `shopId` equality; removed the un-scoped join that caused cross-shop memory access.
- Implemented an orphan-cleanup step in the product update flow: removed image records not present in the new upload array before inserting fresh images.
- Added a publish pre-check: if `shop_products` count is `0` for the shop, the publish request returns `422 Unprocessable Entity` with a message prompting the merchant to add products first.

### Status
**Complete and stable.** The Shop Builder is now a reliable, self-contained merchant onboarding flow.

---



## 3. Backend Hosting — In Progress

### Goal
Deploy the backend to a production-grade host so the application can be submitted to the CKB Nervos community for real-world testing, feedback, and adoption experiments.

### Current Status
- Infrastructure provisioning and environment configuration started.
- Preparing for deployment before community testing begins.

### Next Steps
- Finalize hosting deployment.
- Configure production environment variables and secrets.
- Prepare community submission materials.

---



## 4. Environment Matrix

| Concern | TEST | LIVE |
|---|---|---|
| API Keys | `sk_test_...` / `pk_test_...` | `sk_live_...` / `pk_live_...` |
| Shop subdomain | `-test` suffix reserved | No suffix |
| Webhook signing | Dev secret | Merchant-generated secret |
| AI Model (primary) | `anthropic/claude-haiku-latest` | `anthropic/claude-haiku-latest` |
| AI Model (fallback) | `meta-llama/llama-3.3-70b-instruct:free` | `meta-llama/llama-3.3-70b-instruct:free` |
| Network | Aggron4 (CKB testnet) | Mainnet |

---



## 5. Key Lessons Learned
- Always enforce tenant boundaries (shop, merchant) at the **query level**, not just at the UI or API input level.
- Environment isolation must be continuous — from config load through webhook signing to on-chain transaction submission.
- Edge cases found during "it works" local testing (e.g., empty product lists, subdomain collisions) only surface under real usage scenarios; community testing will likely reveal more.

---



## 6. Community Rollout Readiness

| Item | Status |
|---|---|
| API Key Generation (TEST + LIVE) | Ready |
| Shop Builder end-to-end | Ready |
| Backend hosting | In Progress |
| Community submission | Pending hosting completion |
