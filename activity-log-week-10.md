# Activity Log - Week 10

## Summary
Week 10 examined language choices for CKB smart contracts, balancing production-readiness, developer productivity, security, and performance. The week reviewed recommended production languages, experimental options, and highlighted novel languages and tools (e.g., ZetZ) that enable verification-first, efficient contracts.

## Recommendations: Production vs Experimental
- Production: Historically C; Rust is the primary recommended path moving forward once support stabilizes. Always perform security audits regardless of language.
- High-level productivity option: JavaScript (Duktape) is viable for fast iteration where cycle cost is acceptable and engine security/audit is ensured.
- Experimental: AssemblyScript, Lua, MicroPython, Zig, and other LLVM/WASM-targeting languages are good for prototyping; move to audited languages for production.

## When to Pick Each Language
- Performance- and crypto-heavy primitives: C (hand-optimized, assembly for hot paths) or Rust with careful benchmarking.
- Rapid iteration / non-critical logic: JavaScript or AssemblyScript for developer speed.
- Full Rust ecosystem: WAVM + WASI or native Rust (no_std) paths — choose WAVM/WASI if you need `std` and richer crates.io access.
- Research / formal assurance: Languages with verification features (e.g., ZetZ) for theorem-proven preconditions and safer low-level code.

## Cryptographic Primitives
- Crypto libs commonly use hand-optimized assembly; C and Rust allow low-level tuning and leveraging RISC-V extensions (Vector `V`, Bitmanip `B`) when available.
- For highest performance, hand-written RISC-V assembly remains an option for critical kernels.

## WebAssembly & WASI Implications
- WASM is a portable compilation target; prefer WAVM→RISC-V or WASI paths when you need `std` or rich ecosystem libraries.
- Beware WASM spec fragmentation: pick targets carefully (`wasm32-wasi` preferred for Rust std support).
- CKB's RISC-V base + WASM-on-top approach enables evolving WASM features without chain forks.

## ZetZ and Verified Languages
- ZetZ compiles to C and uses SMT-based verification to assert callsite and buffer invariants at build time.
- The prover prevents a class of runtime errors with zero runtime overhead in the emitted C code.
- Useful where formal guarantees are valuable (IoT, crypto primitives, highly sensitive business logic).

## Practical Guidance for wt-payments-server
- Use Rust for production validation logic once `wasm32-wasi` support is stable; adopt WAVM path for richer Rust std access if needed.
- Keep hot crypto code in C or hand-optimized assembly; measure cycles and fees before moving to high-level languages.
- Use JavaScript for fast iteration in staging or non-cost-sensitive parts; optimize later to bytecode/serialized approaches if performance matters.
- Consider ZetZ for parts that benefit from formal verification (e.g., critical state transitions).

## Activities Completed
- Reviewed `Class 10: Language Choices` doc and recommendations
- Summarized production vs experimental language tradeoffs
- Identified practical next steps for hybrid language strategies in payments infrastructure

## Next Steps
- Prototype a Rust WAVM/WASI validation script for a payment flow and benchmark cycles
- Identify hot crypto kernels to keep in C/assembly
- Experiment with ZetZ on a small, safety-critical validation path

## Resources Reviewed
- CKB language support docs: Rust, JavaScript, WASM paths
- ZetZ documentation and example (carrot demo)
- WASI and WAVM notes for Rust std integration

## Week 10 Takeaway
Choose the language that matches your risk, performance, and developer-velocity tradeoffs: C/Rust for production and crypto-critical code; JavaScript/AssemblyScript for fast iteration; verified languages like ZetZ where formal proofs matter. CKB's flexible VM strategy and WASM support let you mix and match to balance speed, safety, and productivity.
