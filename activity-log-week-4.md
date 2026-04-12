# Activity Log - Week 4

## Summary
Week 4 was a deep dive into getting our hands dirty — this was the most intensive development week so far. Building on the integration groundwork laid in Week 3, the focus shifted entirely to active development, debugging, and pushing toward a working CKB blockchain integration on our payments server project: [wt-payments-server](https://github.com/zenlabs-tech/wt-payments-server.git).

## Development & Debugging

### CKB Integration Progress
- Continued the blockchain integration where we left off in Week 3, now moving beyond planning into active coding and testing.
- Wired up CKB transaction building using Lumos into the payments server API, allowing the server to construct and broadcast on-chain payment transactions.
- Worked on mapping business payment intents (e.g., "Business A receives 500 CKB from Customer X") to actual CKB Cell Model constructs — inputs, outputs, lock scripts, and capacity.

### Debugging Sessions
- Hit a number of integration issues around Lumos indexer sync — transactions were being constructed but the indexer tip was lagging behind the chain tip, causing live cell queries to return stale or empty results. Diagnosed and resolved by adding indexer readiness checks before transaction construction.
- Debugged an issue where Cell capacity was being calculated incorrectly for business payment cells, causing `InsufficientCellCapacity` errors. Fixed by properly accounting for all fields (lock script, type script, data, and capacity bytes) in the capacity calculation.
- Resolved signing flow issues where the transaction witness was not being correctly attached before broadcast, resulting in rejected transactions.
- Tracked down a race condition in the async payment listener where multiple incoming payment requests were colliding on the same unspent cells (double-spend attempts). Addressed with a cell locking/reservation mechanism.

### Crypto Bank Architecture
- Further refined the "crypto bank" architecture of the project: businesses register their CKB address, and the server manages cell tracking, payment verification, and balance aggregation on their behalf.
- Designed a simple payment confirmation flow — once a transaction is confirmed on-chain, the server updates the business's payment record and triggers a webhook notification.

## Activities Completed
- Implemented and tested CKB transaction building and broadcasting within the payments server.
- Resolved multiple critical bugs in the indexer sync, cell capacity calculation, transaction signing, and concurrency handling.
- Completed a working end-to-end test: a simulated business receiving a CKB payment on the devnet, confirmed on-chain, and reflected in the server's records.
- Reviewed and cleaned up the codebase for maintainability going forward.

## Challenges
- The CKB Cell Model's capacity-as-storage constraint added complexity to payment cell design — more care is needed compared to account-model blockchains.
- Async/await patterns with Lumos required careful handling to avoid race conditions in a multi-business payment environment.

## Next Steps
- Finalize and harden the integration for production readiness.
- Add support for multi-business environments with isolated Cell management per business.
- Begin work on the UI/dashboard layer for business owners to monitor incoming payments in real time.
- Write integration tests to cover edge cases in the payment flow.
