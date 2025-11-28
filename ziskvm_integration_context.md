# ZiskVM Integration Context Dump for Erigon FEP

## Date: November 26, 2025
## Status: EXECUTION PLAN COMPLETE

## Full Execution Plan

A comprehensive execution plan has been generated at:
**`/workspace/ZISKVM_FEP_INTEGRATION_PLAN.md`**

## Architecture Decision: Dual-Proof System

```
CDK-Erigon (Execution + Consensus)
    ├─ Witness Generation (stateless proving)
    ├─ Transaction Batching (prover-optimized)
    └─ Two-Tier Proof System:
        ├─ Pessimistic Proofs via SP1 (AggLayer/Bridge) [UNCHANGED]
        │   └─ Bridge state + withdrawal validation
        └─ ZiskVM Execution Proofs (Full verification) [NEW]
            ├─ Block execution proofs
            ├─ Per-transaction proofs (optional)
            └─ State transition verification
```

## Key Architectural Principles

1. **DO NOT REPLACE** SP1-based Pessimistic Proofs (bridge security)
2. **USE ZiskVM** specifically for Full Execution Proofs (FEPs)
3. **Maintain backward compatibility** with legacy executor
4. **Configurable** - toggle between legacy/ZiskVM via flags
5. **Async processing** - maintain non-blocking verification

## Current Architecture Analysis

### Erigon's Current Execution Proof System

1. **LegacyExecutorVerifier** (`zk/legacy_executor_verifier/legacy_executor_verifier.go`)
   - Manages async verification of batch execution
   - Uses gRPC to communicate with remote executor service
   - Sends witness + datastream payload to executor
   - Processes results sequentially via promises

2. **Executor** (`zk/legacy_executor_verifier/executor.go`)
   - gRPC client connecting to executor service
   - Supports multiple concurrent requests
   - Main method: `Verify(payload, request, oldStateRoot)`
   - Uses `ProcessStatelessBatchV2` gRPC call

3. **Witness Generator** (`zk/witness/witness.go`)
   - Generates witness from block execution
   - `GetWitnessByBlockRange()` - main entry point
   - Executes blocks ephemerally and builds SMT witness
   - Uses trie/state operations

4. **Payload Structure**:
   ```go
   type Payload struct {
       Witness         []byte  // SMT partial tree, SCs, old state root
       DataStream      []byte  // txs, batch info, fork id, effective gas
       Coinbase        string  // sequencer address
       OldAccInputHash []byte
       L1InfoRoot      []byte
       TimestampLimit  uint64
       ForcedBlockhashL1 []byte
       ContextId       string
       L1InfoTreeMinTimestamps map[uint64]uint64
   }
   ```

5. **Configuration** (`eth/ethconfig/config_zkevm.go`)
   - `ExecutorUrls` - gRPC URLs for executors
   - `ExecutorEnabled` - toggle executor usage
   - `ExecutorMaxConcurrentRequests`
   - `ExecutorRequestTimeout`
   - `PessimisticForkNumber` - PP fork handling (fork 12+)

### ZiskVM Architecture

1. **Core Concept**: RISC-V based zkVM (riscv64ima-zisk-zkvm-elf)
2. **Programming Model**:
   - Rust programs using `ziskos` crate
   - Input via `ziskos::read_input()` (binary file)
   - Output via `ziskos::set_output(id, value)`
3. **Build**: `cargo-zisk build --release`
4. **Emulation**: `ziskemu` for testing
5. **Proving**: `cargo-zisk prove -e <elf> -i <input> -o <output>`

### ZiskVM Precompiles (Critical for EVM)
- `syscall_keccak_f` - Keccak-f[1600]
- `syscall_sha256_f` - SHA-256
- `syscall_secp256k1_add/dbl` - ECDSA operations
- `syscall_bn254_curve_add/dbl` - BN254 pairing
- `syscall_arith256_mod` - 256-bit modular arithmetic

## Implementation Phases (from full plan)

| Phase | Duration | Focus |
|-------|----------|-------|
| Phase 1 | 2-3 weeks | Foundation & Infrastructure |
| Phase 2 | 6-8 weeks | ZiskVM zkEVM Program (Rust) |
| Phase 3 | 3-4 weeks | CDK-Erigon Integration (Go) |
| Phase 4 | 2-3 weeks | Verification & AggLayer |
| Phase 5 | 3-4 weeks | Optimization & Production |

**Total: 16-24 weeks (4-6 months)**

## Key Files to Modify

| File | Change |
|------|--------|
| `eth/ethconfig/config_zkevm.go` | Add ZiskVM config fields |
| `cmd/utils/flags.go` | Add ZiskVM CLI flags |
| `turbo/cli/flags_zkevm.go` | Apply ZiskVM flags |
| `zk/stages/stage_sequence_execute.go` | Use verifier multiplexer |
| `zk/stages/stage_sequence_execute_utils.go` | Add multiplexer support |

## New Packages to Create

```
zk/
├── ziskvm_verifier/           # NEW
│   ├── verifier.go
│   ├── encoder.go
│   ├── decoder.go
│   ├── service.go
│   ├── proof_store.go
│   ├── agglayer.go
│   └── metrics.go
└── verifier_multiplexer/      # NEW
    ├── multiplexer.go
    └── strategy.go
```

## Critical Integration Point

**`zk/stages/stage_sequence_execute.go` Line 808:**
```go
cfg.legacyVerifier.StartAsyncVerification(...)
```
→ Replace with verifier multiplexer call

## Detailed Plan Reference

See `/workspace/ZISKVM_FEP_INTEGRATION_PLAN.md` for:
- Complete code examples
- Interface definitions
- Configuration schemas
- Testing strategies (TDD-compliant)
- Risk mitigations
- Timeline breakdowns
- CDK-Erigon code standards (ABOUTME, error handling, logging)

## CDK-Erigon Specific Considerations

### Code Standards Required
- **ABOUTME comments**: Every new file must start with 2 ABOUTME comments
- **TDD**: Write failing tests first, then minimal code
- **Test naming**: `Test<Function>_<Scenario>_<ExpectedBehavior>`
- **Coverage**: Minimum 70% line coverage
- **No temporal adjectives**: Avoid "new", "old", "improved" in comments

### Performance Notes
- **SMT/Poseidon**: CPU-intensive, faster on x86 than Apple Silicon
- **Witness Generation**: Can block RPC, use caching
- **Memory**: Monitor during proof generation
- **GPU**: Strongly recommended for ZiskVM proving

### Build Requirements
- **Go version**: 1.24+ (enforced by Makefile)
- **Dependencies**: `make build-libs` for platform-specific deps
- **Test command**: `make test` (10m timeout), `make test-integration` (240m)
- **Lint command**: `make lint`
