# Activity Log - Week 9

## Summary
Week 9 focused on understanding performant WebAssembly (WASM) execution on CKB through direct LLVM compilation, comparing multiple compilation strategies, and exploring WASI support for Rust std library access. The week revealed how CKB's RISC-V foundation enables superior WASM optimization compared to pure WASM engines, and positioned CKB as a blockchain correctly leveraging WASM with architectural foresight.

## Learning & Research

### The WASM Performance Problem
- **Previous Approach**: Translating WASM → C code → RISC-V (via WABT toolchain)
- **Limitations**:
  - Cannot preserve all optimizations through C intermediate layer
  - C limitations prevent full memory layout customization
  - C translation can be flaky and difficult to debug
  - Overhead comes from VM setup and C inefficiencies

### WAVM Solution: Direct LLVM-to-RISC-V Translation
- **Architecture**: WAVM compiles WASM directly to native code via LLVM
- **Key Advantage**: LLVM 9+ has official RISC-V support, enabling direct targeting
- **Runtime Generation**: Custom tool (`wavm-aot-generator`) processes WASM to emit minimal C runtime
- **Result**: Single standalone RISC-V executable compiled from WASM, no external runtime dependency
- **Binary Format**: WAVM generates precompiled objects with native code embedded in custom WASM sections

## Example 1: Fibonacci Comparison

### Compilation Pipeline
1. AssemblyScript source → WASM bytecode
2. WABT path: WASM → C code → compiled RISC-V
3. WAVM path: WASM → precompiled WASM → native RISC-V
4. Native path: Pure C → compiled RISC-V

### Binary Size Comparison
| Version | Size | Analysis |
|---------|------|----------|
| Native C | 11K | Baseline; smallest |
| WABT (C2RV) | 53K | 4.8x larger; includes C runtime |
| WAVM (direct) | 88K | 8x larger; all symbols public (DCE limitation) |

### Performance: Fibonacci(16), Fibonacci(32), Fibonacci(256)
| Metric | WABT | WAVM | Native |
|--------|------|------|--------|
| Setup Cycles | ~542,836 | ~2,530 | ~135 |
| Total Cycles (Fib 16) | 549,478 | 22,402 | 3,114 |
| Total Cycles (Fib 32) | 549,590 | 22,578 | 3,226 |
| Total Cycles (Fib 256) | 551,158 | 25,042 | 4,794 |
| Iteration Cost | ~7 cycles/iter | ~11 cycles/iter | ~7 cycles/iter |

### Key Insights
- **Setup Dominates**: WABT spends 98%+ cycles on VM initialization; WAVM only ~10%
- **WAVM Binary Size**: All symbols public prevents dead code elimination (DCE)
  - AssemblyScript generates less DCE than Rust/LLVM-generated WASM
  - Possible future fix: make non-exported functions module-local for better DCE
- **Per-Iteration Performance**: WAVM slightly slower (~11 vs ~7 cycles) but setup savings dominate
- **Tradeoff**: WAVM wins on setup; native C wins on iteration efficiency

### Setup Cycle Breakdown
- **Native**: ~135 cycles for program initialization
- **WAVM**: ~2,530 cycles (18x native); mostly from memory layout setup
- **WABT**: ~542,836 cycles (4000x native); includes full C runtime initialization, VM setup, memory management

## Example 2: Secp256k1 Cryptographic Operations

### Binary Sizes
- **WABT**: 1,791,744 bytes (~1.7MB)
- **WAVM**: 1,800,440 bytes (~1.8MB)
- Both similar size; WAVM loads slightly fewer bytes into VM

### Performance: Single Operation + 5 Operations
| Test Case | WABT Cycles | WAVM Cycles | Ratio |
|-----------|------------|------------|-------|
| 1x Secp256k1 | 35,702,943 | 10,206,568 | 3.5x faster |
| 5x Secp256k1 | 90,164,183 | 49,307,936 | 1.8x faster |
| Per-Op WABT | ~13,615,310 | — | — |
| Per-Op WAVM | — | ~9,775,342 | — |

### Setup Cost Analysis
- **WABT Setup**: ~21,649,573 cycles (61% of first operation)
- **WAVM Setup**: ~2,462 cycles (0.025% of first operation)
- **Setup Amortization**: On complex operations, WAVM setup becomes negligible

### WAVM Superiority for Complex Code
- LLVM's native code generation better than C restoration for complex logic
- Complex algorithms benefit more than simple ones (compare Fib vs Secp256k1)
- Rust-to-WASM-to-RISC-V chain still viable despite compilation layers

## Example 3: Native C Baseline Reference

### Secp256k1 Pure C Implementation
- **Per-Operation**: 1,346,501 cycles
- **Setup**: 2,463 cycles
- **Binary Size**: Smallest (implied from benchmark set)

### Performance Gap Analysis
- **WAVM vs C**: ~7.3x slower per operation (9.7M vs 1.3M cycles)
- **WABT vs C**: ~10.1x slower per operation (13.6M vs 1.3M)
- **Assessment**: WAVM acceptable for many use cases given setup and language advantages

### Important Caveat
- Rust and C secp256k1 implementations are different algorithms
- Performance comparison is approximate; actual quality depends on specific implementations
- Rust version includes bound checking not present in C version

## WASI Support: Bringing Rust std to CKB

### The Problem
- Pure Rust → RISC-V path requires `no_std` (core-only Rust)
- Rust std is not available on RISC-V; libc bindings absent
- Limits access to most crates.io libraries and Rust ecosystem

### The Solution: WASI (WebAssembly System Interface)
- **Standard Interface**: WASI provides standardized way to interface WASM with environment
- **Rust Alignment**: Rust community standardized on `wasm32-wasi` target for WASM future
- **WAVM Support**: WAVM already implements WASI API shims
- **CKB Integration**: WAVM's WASI layer can be extended to CKB syscalls

### Hello World WASI Example
```
WASM (WASI APIs) → WAVM precompile → glue code + runtime → RISC-V executable
```

- Example runs WASI hello world on CKB
- Output: `Hello World!` printed via CKB debug syscall
- Setup: ~2,532 cycles (minimal overhead)
- Proof: WASI APIs work correctly with CKB integration

### Future Roadmap
- Extend WASI shim implementations for all WASI APIs
- Enable Rust programs with std library support compiled to `wasm32-wasi`
- Full Rust ecosystem access for CKB smart contracts
- Future-proof Rust development experience

## Architectural Philosophy: RISC-V vs WASM as Base Layer

### Why RISC-V is Superior for Blockchain
**RISC-V Advantages**:
- Designed for hardware; spec fixed forever
- Simple specification; minimal feature additions
- Instruction set maps directly to CPU instructions
- Direct CPU instruction mapping enables secure minimal layer

**WASM Evolution Problem**:
- Living spec; new features constantly added (memory.grow, callIndirect, garbage collection, threading)
- Blockchain must choose which WASM feature set to support
- Future features have no direct CPU mappings
- Spec changes force incompatible migrations or hard forks

### The RISC-V + WASM Solution
- **RISC-V Foundation**: Fixed, unchanging instruction set
- **WASM on Top**: Implement WASM as application layer on RISC-V
- **Benefits**:
  - Can update WASM implementation without chain consensus
  - Can adopt new WASM features via smart contract upgrades
  - No forks needed for feature additions
  - Example: Discover new garbage collection → deploy as contract upgrade

### Comparison: Blockchain WASM Implementations
**Problem with Direct WASM Engines**:
- Most blockchains claim WASM support but choose minimal subset
- Limited to WASM features (no std, no full Rust)
- Many only support deprecated `wasm32-unknown-unknown` target
- Cannot easily adopt new WASM features
- Future-incompatible once features change

**CKB's Approach**:
- RISC-V is the fixed foundation
- WASM is optional layer
- Rust std is available via `wasm32-wasi` target
- Full WASI roadmap planned
- Future-proof architecture

## Performance Summary Table

| Benchmark | Best | Medium | Worst | Notes |
|-----------|------|--------|-------|-------|
| **Fibonacci** | Native C (3K cycles) | WAVM (22K cycles) | WABT (549K cycles) | Setup dominates; iteration similar |
| **Secp256k1** | Native C (1.6M cycles) | WAVM (10M cycles) | WABT (35M cycles) | Complex code; WAVM/LLVM shines |
| **Setup Cost** | Native (135) | WAVM (2.5K) | WABT (542K) | WAVM 18x native; WABT 4000x native |
| **Binary Size** | Native (11K) | WABT (1.7M) | WAVM (1.8M) | For crypto: sizes similar |

## Key Technical Insights

| Concept | Key Point |
|---------|-----------|
| WAVM Direct Translation | LLVM compiles WASM→RISC-V without C intermediate |
| Setup Amortization | Fixed setup cost decreases as percentage on longer operations |
| DCE Limitation | Public symbols prevent dead code elimination; affects size |
| LLVM Quality | LLVM→RISC-V sometimes better than C→RISC-V for complex code |
| Secp256k1 7x Gap | Rust WASM vs C: acceptable overhead for ecosystem benefits |
| WASI Viability | WASI APIs work on CKB; enables full Rust std in future |
| Binary Patcher | Workaround for VM bug; patches LLVM-generated RISC-V |

## Connection to CKB Builder Ecosystem

### Implications for Payment Server
1. **Validation Scripts**: Can now write in Rust with std library access
2. **Performance**: 7-10x slower than C acceptable for validation (not tight loops)
3. **Ecosystem**: Access to full crates.io for business logic
4. **Maintenance**: Easier to debug and maintain Rust code vs C
5. **Future**: WASI roadmap means better performance and more features over time

### Strategic Advantage
- **Language Flexibility**: C for performance-critical sections; Rust for logic
- **Ecosystem Access**: Leverage Rust community libraries and patterns
- **Future-Proof**: WASI adoption means long-term viability
- **Blockchain Philosophy**: RISC-V+WASM correctly implements blockchain fundamentals

## Activities Completed
- Studied WAVM architecture and LLVM-to-RISC-V compilation
- Analyzed Fibonacci benchmark across three implementations
- Traced secp256k1 performance and setup cost amortization
- Reviewed binary size tradeoffs (native vs WAVM vs WABT)
- Explored WASI support and Rust std library access
- Tested Hello World WASI example on CKB
- Reviewed RISC-V vs WASM philosophical comparison
- Analyzed blockchain WASM implementation patterns

## Conceptual Breakthroughs
- **Platform-Aware Optimization**: CKB's fixed RISC-V foundation enables optimizations impossible on evolving WASM engines
- **Future Immunity**: RISC-V foundation means no forks for future WASM features
- **Ecosystem Bridge**: WASM→RISC-V enables full Rust ecosystem without compromising blockchain simplicity
- **Setup Amortization**: Even 18x overhead is acceptable for setup costs that decrease as percentage on real workloads
- **Blockchain Design Philosophy**: Most blockchains use WASM incorrectly; CKB uses it correctly
- **Ecosystem vs Performance**: 7x performance overhead justified by Rust std access and maintainability

## Next Steps
- Benchmark payment validation scripts written in Rust vs C
- Explore WASI API extensions for CKB syscalls
- Test complex business logic in optimized WAVM Rust
- Design hybrid approach: C for crypto, Rust for orchestration
- Monitor WAVM and LLVM improvements for future performance gains
- Consider WASI adoption timeline for Rust std availability
- Evaluate crates for payment domain (cryptography, serialization, etc.)

## Resources Reviewed
- WAVM Architecture and LLVM Integration
- WebAssembly Binary Toolkit (WABT) Comparison
- WAVM-AOT-Generator for RISC-V Runtime
- CKB Binary Patcher for RISC-V VM Bug Workaround
- WASI (WebAssembly System Interface) Specification
- Rust wasm32-wasi Target and std Library Support
- CKB Standalone Debugger Benchmarking
- RISC-V vs WASM Architectural Philosophy

## Week 9 Takeaway
CKB correctly leverages WebAssembly by building it on top of a fixed RISC-V foundation, not as the base layer. This enables WASM feature adoption without blockchain forks, Rust std library access through WASI, and language choice flexibility (C for performance, Rust for ecosystem). While WAVM-compiled scripts consume 7-10x more cycles than pure C, the benefit of Rust ecosystem access and maintainability makes this tradeoff worthwhile for most use cases. Future WASI extensions will narrow this gap while RISC-V's fixed specification ensures long-term compatibility and evolution capability.
