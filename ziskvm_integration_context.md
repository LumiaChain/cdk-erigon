# ZiskVM Integration Context Dump for Erigon FEP

## Date: November 26, 2025

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

## Integration Points Identified

1. **Verifier Interface** - Replace `LegacyExecutorVerifier` with ZiskVM-based verifier
2. **Witness Format** - Transform SMT witness to ZiskVM input format
3. **EVM Program** - Need zkEVM implementation as ZiskVM program
4. **Proof Output** - Adapt verification of ZiskVM proofs
5. **Configuration** - New config options for ZiskVM

## Key Files to Modify/Create
- `zk/ziskvm_verifier/` - New verifier package
- `eth/ethconfig/config_zkevm.go` - Add ZiskVM config
- `zk/stages/stage_sequence_execute.go` - Update verifier calls
- `turbo/cli/flags_zkevm.go` - Add CLI flags
