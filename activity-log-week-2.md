## Evidence: Watched Workshop Videos

The following YouTube videos were watched and summarized as part of this week's activities:

- https://youtu.be/iVjccs3z5q0?si=7IGvNMD0PWHD9w8I
- https://youtu.be/NcN3NiBuJbo?si=xpd93MjJ1xlYuEp3
- https://youtu.be/lOxXrVIfT2Y?si=ZGqncHK1PYqr1th_
- https://youtu.be/TJ2bnSFUpPQ?si=6pYtNf-Ap8GYFdWW
# Activity Log - Week 2

## Summary
This week, we continued our journey in NFT development on the Nervos CKB platform. The main activities included:

- Summarized key learnings from the CKB Academy's "Getting Started With NFTs" guide, focusing on:
  - Introduction to NFTs on CKB and Godwoken (L1 and L2)
  - Comparison of NFT standards: CoTA (low cost, SMT-based, account model) and Spore (full on-chain content, cell model, strong permanence)
  - Benefits and trade-offs of each standard for different use cases
  - Importance of state rent, permanence, and cost considerations in NFT design
- Started the L1 Developer Training Course to deepen understanding of the Cell Model and smart contract development on CKB.
- Cloned the official developer training course repository:
  - Repository: https://github.com/jordanmack/developer-training-course.git
- Set up the lab environment for hands-on practice with CKB smart contracts and NFT standards.

## Key Takeaways
- CoTA is ideal for low-cost, high-volume NFT use cases.
- Spore is best for NFTs requiring strong permanence and full on-chain data storage.
- Nervos CKB's template-based smart contracts allow developers to use existing deployments, saving costs and improving sustainability.
- Understanding the Cell Model is essential for advanced NFT and smart contract development on CKB.



## CKB Dapps Workshop Video Summaries

### Workshop Part 1: CKB Dapps Overview
This week, I watched the "Dapps with CKB Workshop" by Xuejie from Nervos Foundation. The workshop provided a comprehensive introduction to developing decentralized applications (Dapps) on the Nervos CKB blockchain, covering both on-chain and off-chain development, and highlighting the unique features of CKB compared to other blockchains like Ethereum and Bitcoin.

**Key Points:**
- CKB (Common Knowledge Base) is the Layer 1 blockchain in the Nervos ecosystem, designed for security and long-term flexibility.
- CKB extends Bitcoin's UTXO model into the more flexible Cell Model, where each transaction consumes and creates cells that can store value, arbitrary data, and scripts (lock and type scripts).
- Lock scripts control ownership, while type scripts enforce custom logic (e.g., token rules).
- Unlike Ethereum's account/world state model, CKB's state is always the set of unspent cells, making it deterministic and immutable after transaction inclusion.
- CKB-VM is a virtual machine based on the RISC-V instruction set, supporting languages like Rust, C, and C++.
- CKB-VM allows custom cryptography and even supports compiling other VMs (e.g., EVM, WASM) to run on CKB.
- Common patterns include token transfers, Simple UDT (User Defined Token), Anyone-Can-Pay (ACP) lock, timestamp and time control, key-value storage with sparse Merkle trees, and open transactions for composability.
- CKB contracts can only validate state, not modify it directly; all changes are made via transactions. Only data requiring global consensus should be stored on-chain, with the rest handled off-chain for scalability.

### Workshop Part 2: On-Chain Scripts and Capsule
I also watched the second lecture, which focused on writing on-chain scripts in Rust for CKB using the Capsule tool.

**Key Learnings:**
- The script validation workflow in CKB: For a transaction (tx) to be committed, all lock scripts in input cells and all type scripts in both input and output cells must be executed and validated.
- Lock scripts are mainly for ownership and signature verification, while type scripts implement Dapp logic (e.g., UDTs, NFTs, data append-only rules).
- Scripts are executed in groups (script groups), and each script only processes the cells that use it.
- The script structure includes `code_hash`, `hash_type`, and `args`. The actual script code is stored in a cell and referenced via Cell Dep in the transaction.
- Code reuse is maximized: one on-chain script can be used for many tokens or lock types by varying the args.
- Type scripts have two main responsibilities: identification (so Dapps can recognize cell types) and security (enforcing rules, e.g., no unauthorized token minting).
- The Type ID pattern ensures unique type scripts/cells on-chain, preventing forgery and enabling unique NFTs or contract instances.
- Capsule is a Rust-based framework for developing, testing, and deploying CKB scripts. It uses standard Rust tooling and provides utilities for syscalls, iterators, and testing.
- The workshop included a step-by-step demo of implementing an NFT validator script in Rust, with unit tests and deployment using Capsule.

**Activities Completed:**
- Watched and summarized both CKB Dapps Workshop videos (introduction and on-chain script development).
- Learned about the Cell Model, script validation, script/code structure, and the Capsule toolchain for Rust smart contract development.
- Understood the process of writing, testing, and deploying a CKB on-chain script, including NFT logic and unique type IDs.

### Workshop Part 3: Lumos and NFT Dapp Demo
I also watched the third lecture in the Dapps with CKB Workshop series, which focused on building a simple NFT Dapp using the Lumos framework and TypeScript.

**Key Learnings:**
- The demo code is available in the nervosnetwork/dapps-on-ckb-workshop-code repository, with the on-chain script in `nft-validator` and the Dapp skeleton in `nft-glue`.
- The Dapp uses three test accounts (Alice, Bob, Charlie) for demonstration, with private keys included for local testing only.
- The code demonstrates initializing Lumos, setting up the indexer, and configuring the environment for CKB development.
- Core functions include generating NFT tokens, listing NFTs, and transferring NFTs between accounts, all using transaction skeletons in Lumos.
- The governance lock concept is used to distinguish NFT types (e.g., different event tickets), and the NFT ID is derived from the first input cell hash.
- The demo walks through mining CKB, transferring capacity, deploying the NFT script with Capsule, updating Lumos config, and running the full NFT lifecycle: minting, listing, and transferring NFTs.
- The process includes signing transactions with the appropriate private key and committing them to the chain.
- The video also introduces advanced concepts like fixedEntry in Lumos (to prevent transaction optimization from removing necessary cells), and discusses the flexibility of CKB for Dapp development.
- Additional projects mentioned: keyper (for wallet/Dapp decoupling), pw-core (Ethereum wallet integration), Polyjuice (EVM compatibility), and ckb-simple-account-layer (account model abstraction).

**Activities Completed:**
- Watched and summarized the third Dapps with CKB Workshop video (Lumos/NFT Dapp demo).
- Learned how to use Lumos and Capsule to build, deploy, and interact with NFT Dapps on CKB.
- Practiced the full workflow: mining, account setup, script deployment, config updates, NFT minting, listing, and transfer.

## Next Steps
- Continue progressing through the L1 Developer Training Course.
- Practice writing and deploying CKB scripts using Capsule.
- Explore more advanced Dapp patterns and participate in hands-on exercises with CoTA, Spore, and custom scripts.
