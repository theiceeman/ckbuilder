# Activity Log – Week 1
## Name 
okorie Ebube
 ## Week Ending 
 3/20/2026
## 1. Environment Setup
- Initialized a new workspace for CKB development.
- Prepared to build a CKB transfer dApp using the CCC SDK.
- Verified the workspace structure and ensured the .github/copilot-instructions.md file exists for project guidance.
- Confirmed the devnet node and RPC proxy server are running locally (http://127.0.0.1:28114).
- Noted a Node.js deprecation warning (util._extend), which does not block progress.
- Clarified that the CKB RPC server only accepts POST/OPTIONS requests, not GET (browser navigation to the endpoint is not supported).

## 2. Documentation Study
- Read and summarized CKB basic theoretical knowledge:
  - **What is CKB?**
    - CKB is a blockchain based on the concept of “cells,” similar to Bitcoin’s UTXO model. State changes are represented by spending and creating cells.
    - Cells can store value and arbitrary data, making the model flexible for smart contracts.
  - **What does a cell contain?**
    - Each cell has: capacity (storage size in CKB tokens), lock (ownership script), type optional script for logic, and data arbitrary bytes.
    - The total size of all fields must not exceed the cell’s capacity.
    - Capacity is measured in shannons (1 CKB = 10^8 shannons).
  - **How do you own a cell?**
    - Ownership is determined by the lock script, which uses a code hash and arguments often a public key.
    - To spend a cell, a valid proof (like a signature) must unlock the script.
    - Lock scripts are essential for security and control of assets.

## 3. Key Learnings
- CKB’s state is managed through cells, which can store both value and data.
- Capacity (CKB tokens) equals storage space on-chain.
- Lock scripts are critical for security and ownership.
- The model supports flexible smart contract logic.



## 4. Practical Operation: Wallet, Cells, and Blocks
- Connected to the CKB testnet using  MetaMask .
- Upon connection, received 10 live cells, each with a capacity of 100 CKB (0x2540be400 shannons).
- Used the 'Fetch Cells' feature to refresh and view the list of live cells owned by the wallet.
- Observed that the wallet’s CKB balance is the sum of the capacities of all live cells it can unlock—this represents both value and available on-chain storage.
- Explored the latest blocks on the CKB testnet, viewing block numbers, hashes, timestamps, and transaction counts.
- Clicked on transactions and cells to inspect their detailed JSON structure, noting that real transaction structures are more complex but still follow the input → output model.
- Reviewed the testnet chain configuration, including the address prefix (ckt) and built-in script contracts (SECP256K1_BLAKE160, MULTISIG, DAO, SUDT, ANYONE_CAN_PAY, OMNILOCK).
- Learned that SECP256K1_BLAKE160 is the default lock script for cell ownership, and that built-in scripts are pre-deployed in the testnet genesis block.

## 5. Transaction Practice: Manual Transfer and Signing
- Connected MetaMask wallet to the CKB testnet.
- Practiced constructing a transfer transaction manually:
  - Used the toolbox to find live cells and chain configuration details.
  - Filled in transaction fields, including cellDeps for both SECP256K1_BLAKE160 and OMNILOCK scripts as Omnilock protected the live cells.
  - Entered the correct outPoint (txHash and index) and depType for each dependency.
- Generated the transaction hash (tx_hash) from the filled transaction template, confirming the hash can be known before signing.
- Learned about the witnesses field and the WitnessArgs structure, which holds the signature and optional type script arguments.
- Used the provided tools to:
  - Generate the signing message for the transaction.
  - Sign the transaction using the connected wallet MetaMask.
  - Insert the signature into the witnesses field by serializing WitnessArgs.
- Finalized the transaction and sent it to the CKB testnet.
- Verified that the returned tx_hash matched the pre-signing hash, demonstrating CKB’s transaction determinism.
- Checked the transaction status and block inclusion on-chain.
- Successfully generated and submitted a transaction on the CKB testnet.
- Transaction hash (tx_hash): 0x63c150ecc0f50feb16552b278a4dcb0dfd70d7576539851c4a9fcf6d0e3caa02
- Verified the transaction on the Nervos CKB Testnet Explorer:
  - URL: https://testnet.explorer.nervos.org/transaction/0x63c150ecc0f50feb16552b278a4dcb0dfd70d7576539851c4a9fcf6d0e3caa02
- Observed transaction status and details on the explorer, confirming successful broadcast to the testnet.


