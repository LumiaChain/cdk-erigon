# ZiskVM Full Execution Proofs Integration Plan for CDK-Erigon

## Executive Summary

This document outlines a comprehensive plan to integrate **ZiskVM** for generating **Full Execution Proofs (FEPs)** within the CDK-Erigon codebase, while **preserving the existing SP1-based Pessimistic Proof system** for AggLayer/Bridge security.

### Key Architecture Decision

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

---

## Table of Contents

1. [Current Architecture Analysis](#1-current-architecture-analysis)
2. [ZiskVM Architecture Overview](#2-ziskvm-architecture-overview)
3. [Integration Strategy](#3-integration-strategy)
4. [Phase 1: Foundation & Infrastructure](#4-phase-1-foundation--infrastructure)
5. [Phase 2: ZiskVM zkEVM Program Development](#5-phase-2-ziskvm-zkevm-program-development)
6. [Phase 3: CDK-Erigon Integration](#6-phase-3-cdk-erigon-integration)
7. [Phase 4: Verification & AggLayer Integration](#7-phase-4-verification--agglayer-integration)
8. [Phase 5: Optimization & Production Readiness](#8-phase-5-optimization--production-readiness)
9. [File Modification Matrix](#9-file-modification-matrix)
10. [New Components to Create](#10-new-components-to-create)
11. [Testing Strategy](#11-testing-strategy)
12. [Risks and Mitigations](#12-risks-and-mitigations)
13. [Timeline Estimates](#13-timeline-estimates)
14. [Appendices](#14-appendices)

---

## 1. Current Architecture Analysis

### 1.1 Existing Proof Generation Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CDK-Erigon Sequencer                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐ │
│  │ Stage Sequence  │───>│ Witness Generator│───>│ Legacy Executor     │ │
│  │ Execute         │    │ (SMT Partial Tree│    │ Verifier (gRPC)     │ │
│  └─────────────────┘    │  + DataStream)   │    └─────────────────────┘ │
│          │              └──────────────────┘              │              │
│          │                                                │              │
│          v                                                v              │
│  ┌─────────────────┐                        ┌─────────────────────────┐ │
│  │ DataStream      │                        │ External Executor       │ │
│  │ Server          │                        │ Service (ProcessBatchV2)│ │
│  └─────────────────┘                        └─────────────────────────┘ │
│                                                          │              │
│                                                          v              │
│                                             ┌─────────────────────────┐ │
│                                             │ State Root Verification │ │
│                                             └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Key Components

#### 1.2.1 LegacyExecutorVerifier (`zk/legacy_executor_verifier/legacy_executor_verifier.go`)

**Purpose**: Manages asynchronous batch verification via promises.

**Key Structures**:
```go
type LegacyExecutorVerifier struct {
    db                     kv.RwDB
    cfg                    ethconfig.Zk
    executors              []*Executor         // gRPC executor clients
    streamServer           server.DataStreamServer
    WitnessGenerator       WitnessGenerator
    promises               []*Promise[*VerifierBundle]
    mtxPromises            *sync.Mutex
}

type VerifierRequest struct {
    BatchNumber  uint64
    BlockNumbers []uint64
    ForkId       uint64
    StateRoot    common.Hash
    Counters     map[string]int
}

type VerifierResponse struct {
    Valid            bool
    Witness          []byte
    ExecutorResponse *executor.ProcessBatchResponseV2
    Error            error
}
```

**Critical Methods**:
- `StartAsyncVerification()` - Entry point called from sequencer stage
- `VerifyAsync()` - Sends batch to external executor via gRPC
- `ProcessResultsSequentially()` - Collects verification results

#### 1.2.2 Executor (`zk/legacy_executor_verifier/executor.go`)

**Purpose**: gRPC client for external executor service.

**Payload Structure** (critical for ZiskVM input):
```go
type Payload struct {
    Witness                 []byte   // SMT partial tree + smart contracts + old state root
    DataStream              []byte   // txs, batch info, fork id, effective gas, block header
    Coinbase                string   // sequencer address
    OldAccInputHash         []byte   // 0 for executor, required for prover
    L1InfoRoot              []byte   // 0 for executor, required for prover
    TimestampLimit          uint64   // timestamp constraint
    ForcedBlockhashL1       []byte   // for forced batches
    ContextId               string   // batch identifier
    L1InfoTreeMinTimestamps map[uint64]uint64
}
```

#### 1.2.3 Witness Generator (`zk/witness/witness.go`)

**Purpose**: Generates execution witness by re-executing blocks ephemerally.

**Key Method**:
```go
func (g *Generator) GetWitnessByBlockRange(
    tx kv.Tx, 
    ctx context.Context, 
    startBlock, endBlock uint64, 
    debug, witnessFull bool,
) ([]byte, error)
```

**Process**:
1. Loads blocks from database
2. Unwinds state if necessary
3. Re-executes transactions with state tracking
4. Builds SMT witness from accessed state
5. Serializes witness for prover consumption

#### 1.2.4 Sequencer Integration Point (`zk/stages/stage_sequence_execute.go`)

**Critical Call Site** (Line 808):
```go
cfg.legacyVerifier.StartAsyncVerification(
    batchContext.s.LogPrefix(),
    batchState.forkId,
    batchState.batchNumber,
    block.Root(),
    counters.UsedAsMap(),
    batchState.builtBlocks,
    useExecutorForVerification,
    batchContext.cfg.zk.SequencerBatchVerificationTimeout,
    batchContext.cfg.zk.SequencerBatchVerificationRetries,
)
```

### 1.3 Configuration (`eth/ethconfig/config_zkevm.go`)

**Relevant Fields**:
```go
type Zk struct {
    ExecutorUrls                      []string      // gRPC URLs
    ExecutorEnabled                   bool          // toggle verification
    ExecutorStrictMode                bool          // require executor
    ExecutorMaxConcurrentRequests     int           // parallelism
    ExecutorRequestTimeout            time.Duration // timeout
    WitnessFull                       bool          // full vs partial witness
    WitnessMemdbSize                  datasize.ByteSize
    MockWitnessGeneration             bool          // testing
    PessimisticForkNumber             uint64        // PP fork handling
}
```

### 1.4 Pessimistic Proof Status

The `PessimisticForkNumber` field indicates networks running with Pessimistic Proofs (fork 12+). These proofs are handled **externally via SP1** and should **NOT** be replaced. Our ZiskVM integration focuses solely on Full Execution Proofs (FEPs).

---

## 2. ZiskVM Architecture Overview

### 2.1 Core Concepts

ZiskVM is a high-performance zkVM based on **RISC-V** architecture (`riscv64ima-zisk-zkvm-elf`), designed for generating zero-knowledge proofs of arbitrary program execution.

**Key Technologies**:
- **Plonky3**: STARK-based proof system
- **RISC-V ISA**: Standard instruction set
- **GPU/MPI Acceleration**: Distributed proving

### 2.2 Programming Model

```rust
// ZiskVM Program Structure
#![no_main]
#![no_std]

use ziskos::io::{read_input, set_output};

#[no_mangle]
pub extern "C" fn main() {
    // Read witness + datastream as binary input
    let input: MyInputType = read_input();
    
    // Execute EVM state transition
    let result = execute_state_transition(input);
    
    // Output new state root for verification
    set_output(0, result.new_state_root);
}
```

### 2.3 Critical Precompiles for EVM

ZiskVM provides optimized syscalls essential for EVM compatibility:

| Precompile | Function | Use Case |
|-----------|----------|----------|
| `syscall_keccak_f` | Keccak-f[1600] | Address derivation, storage keys |
| `syscall_sha256_f` | SHA-256 | Data hashing |
| `syscall_secp256k1_add/dbl` | ECDSA operations | Transaction signature verification |
| `syscall_bn254_curve_add/dbl` | BN254 pairing | Precompile 0x06, 0x07, 0x08 |
| `syscall_arith256_mod` | 256-bit modular arithmetic | BigInt operations |
| `syscall_arith384_mod` | 384-bit modular arithmetic | Extended math |

### 2.4 CLI Tools

```bash
# Build ZiskVM program
cargo-zisk build --release

# Run in emulator (testing)
ziskemu -e <elf> -i <input>

# Generate proof
cargo-zisk prove -e <elf> -i <input> -o <proof>

# Verify proof
cargo-zisk verify -e <elf> -p <proof>
```

---

## 3. Integration Strategy

### 3.1 Architecture Design

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         CDK-Erigon with ZiskVM                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌─────────────────┐    ┌──────────────────────────────────────────────────┐ │
│  │ Stage Sequence  │───>│            Verifier Multiplexer                  │ │
│  │ Execute         │    │  ┌─────────────────────────────────────────────┐ │ │
│  └─────────────────┘    │  │                                             │ │ │
│          │              │  │  ┌───────────────┐   ┌──────────────────┐  │ │ │
│          │              │  │  │ Legacy        │   │ ZiskVM           │  │ │ │
│          │              │  │  │ Executor      │   │ Verifier         │  │ │ │
│          │              │  │  │ (gRPC)        │   │ (Native/Service) │  │ │ │
│          │              │  │  └───────────────┘   └──────────────────┘  │ │ │
│          │              │  │         │                   │              │ │ │
│          v              │  └─────────│───────────────────│──────────────┘ │ │
│  ┌─────────────────┐    └────────────│───────────────────│────────────────┘ │
│  │ Witness         │                 │                   │                  │
│  │ Generator       │─────────────────┼───────────────────┘                  │
│  └─────────────────┘                 │                                      │
│          │                           │                                      │
│          │              ┌────────────v────────────┐                         │
│          │              │ Proof Aggregator        │                         │
│          │              │ (FEP + Optional PP)     │                         │
│          │              └────────────┬────────────┘                         │
│          │                           │                                      │
│          v                           v                                      │
│  ┌─────────────────┐    ┌─────────────────────────┐                         │
│  │ DataStream      │    │ AggLayer Submission     │                         │
│  │ Server          │    │ (FEP proofs)            │                         │
│  └─────────────────┘    └─────────────────────────┘                         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Integration Modes

**Mode 1: Sidecar Service** (Recommended for Production)
- ZiskVM runs as a separate gRPC/JSON-RPC service
- Erigon sends witness + datastream, receives proof
- Supports horizontal scaling and GPU acceleration

**Mode 2: Native Integration** (Development/Testing)
- ZiskVM proving embedded in Go via CGO or subprocess
- Lower latency for small batches
- Simpler deployment

**Mode 3: Hybrid**
- Local verification for testing
- Remote proving for production

### 3.3 Key Design Principles

1. **Non-Breaking**: Maintain backward compatibility with legacy executor
2. **Configurable**: Toggle between legacy and ZiskVM via flags
3. **Dual-Proof**: Support both FEP (ZiskVM) and PP (SP1) simultaneously
4. **Witness Reuse**: Same witness format for both systems where possible
5. **Async Processing**: Maintain non-blocking verification

---

## 4. Phase 1: Foundation & Infrastructure

### 4.1 Objectives

- [ ] Set up ZiskVM development environment
- [ ] Create ZiskVM verifier interface in Erigon
- [ ] Define input/output encoding formats
- [ ] Establish project structure

### 4.2 Tasks

#### 4.2.1 Create ZiskVM Verifier Package Structure

```
zk/
├── legacy_executor_verifier/     # Existing (unchanged)
├── ziskvm_verifier/              # NEW
│   ├── verifier.go               # Main verifier implementation
│   ├── encoder.go                # Input encoding (witness → ZiskVM format)
│   ├── decoder.go                # Output decoding (proof → verification result)
│   ├── service.go                # gRPC/HTTP service client
│   ├── config.go                 # ZiskVM-specific configuration
│   ├── proof.go                  # Proof structures
│   ├── precompiles.go            # Precompile mappings
│   └── proto/
│       └── ziskvm_service.proto  # Service definitions
└── verifier_multiplexer/         # NEW
    ├── multiplexer.go            # Routes to appropriate verifier
    └── strategy.go               # Selection logic
```

#### 4.2.2 Define Core Interfaces

```go
// zk/ziskvm_verifier/interfaces.go

package ziskvm_verifier

import (
    "context"
    "github.com/erigontech/erigon-lib/common"
)

// ZiskVMVerifier defines the interface for ZiskVM-based verification
type ZiskVMVerifier interface {
    // VerifyBatch generates a ZK proof for a batch and verifies it
    VerifyBatch(ctx context.Context, req *VerificationRequest) (*VerificationResult, error)
    
    // GenerateProof generates a ZK proof without verification
    GenerateProof(ctx context.Context, req *VerificationRequest) (*ZiskProof, error)
    
    // VerifyProof verifies an existing proof
    VerifyProof(ctx context.Context, proof *ZiskProof) (bool, error)
    
    // Health check
    CheckOnline() bool
    
    // Cleanup
    Close() error
}

// VerificationRequest contains all data needed for batch verification
type VerificationRequest struct {
    BatchNumber     uint64
    ForkId          uint64
    BlockNumbers    []uint64
    
    // Witness data (SMT partial tree)
    Witness         []byte
    
    // DataStream (transactions, headers, etc.)
    DataStream      []byte
    
    // Expected state root for verification
    ExpectedStateRoot common.Hash
    OldStateRoot      common.Hash
    
    // L1 info tree data
    L1InfoRoot        []byte
    L1InfoTreeMinTimestamps map[uint64]uint64
    
    // Metadata
    Coinbase          common.Address
    TimestampLimit    uint64
    ContextId         string
}

// VerificationResult contains the outcome of verification
type VerificationResult struct {
    Valid            bool
    NewStateRoot     common.Hash
    Proof            *ZiskProof
    ProverMetrics    *ProverMetrics
    Error            error
}

// ZiskProof represents a ZiskVM-generated proof
type ZiskProof struct {
    ProofBytes       []byte
    PublicInputs     []byte
    VerificationKey  []byte
    BatchNumber      uint64
    Timestamp        int64
}

// ProverMetrics contains performance data
type ProverMetrics struct {
    ProvingTimeMs    int64
    VerifyTimeMs     int64
    WitnessSizeBytes int64
    ProofSizeBytes   int64
    Cycles           uint64
}
```

#### 4.2.3 Configuration Extensions

```go
// eth/ethconfig/config_zkevm.go - ADDITIONS

type Zk struct {
    // ... existing fields ...
    
    // ZiskVM Configuration
    ZiskVMEnabled                    bool          `yaml:"zkevm.ziskvm-enabled"`
    ZiskVMServiceUrls                []string      `yaml:"zkevm.ziskvm-service-urls"`
    ZiskVMProgramPath                string        `yaml:"zkevm.ziskvm-program-path"`
    ZiskVMProvingKeyPath             string        `yaml:"zkevm.ziskvm-proving-key-path"`
    ZiskVMVerificationKeyPath        string        `yaml:"zkevm.ziskvm-verification-key-path"`
    ZiskVMMaxConcurrentRequests      int           `yaml:"zkevm.ziskvm-max-concurrent-requests"`
    ZiskVMRequestTimeout             time.Duration `yaml:"zkevm.ziskvm-request-timeout"`
    ZiskVMProofOutputPath            string        `yaml:"zkevm.ziskvm-proof-output-path"`
    ZiskVMUseGPU                     bool          `yaml:"zkevm.ziskvm-use-gpu"`
    ZiskVMMPIEnabled                 bool          `yaml:"zkevm.ziskvm-mpi-enabled"`
    ZiskVMEmulatorMode               bool          `yaml:"zkevm.ziskvm-emulator-mode"`
    
    // Proof type selection
    PreferZiskVMForFEP               bool          `yaml:"zkevm.prefer-ziskvm-for-fep"`
    FallbackToLegacyOnZiskVMFailure  bool          `yaml:"zkevm.fallback-to-legacy-on-ziskvm-failure"`
}

// Helper methods
func (c *Zk) UseZiskVM() bool {
    return c.ZiskVMEnabled && len(c.ZiskVMServiceUrls) > 0
}

func (c *Zk) HasZiskVMService() bool {
    return len(c.ZiskVMServiceUrls) > 0 && c.ZiskVMServiceUrls[0] != ""
}
```

#### 4.2.4 CLI Flags

```go
// cmd/utils/flags.go - ADDITIONS

var (
    ZiskVMEnabled = cli.BoolFlag{
        Name:  "zkevm.ziskvm-enabled",
        Usage: "Enable ZiskVM for Full Execution Proof generation",
        Value: false,
    }
    
    ZiskVMServiceUrls = cli.StringFlag{
        Name:  "zkevm.ziskvm-service-urls",
        Usage: "Comma-separated list of ZiskVM prover service URLs",
        Value: "",
    }
    
    ZiskVMProgramPath = cli.StringFlag{
        Name:  "zkevm.ziskvm-program-path",
        Usage: "Path to compiled ZiskVM zkEVM program (ELF)",
        Value: "",
    }
    
    ZiskVMProvingKeyPath = cli.StringFlag{
        Name:  "zkevm.ziskvm-proving-key-path",
        Usage: "Path to ZiskVM proving key",
        Value: "",
    }
    
    ZiskVMVerificationKeyPath = cli.StringFlag{
        Name:  "zkevm.ziskvm-verification-key-path",
        Usage: "Path to ZiskVM verification key",
        Value: "",
    }
    
    ZiskVMRequestTimeout = cli.DurationFlag{
        Name:  "zkevm.ziskvm-request-timeout",
        Usage: "Timeout for ZiskVM proof generation requests",
        Value: 10 * time.Minute,
    }
    
    ZiskVMEmulatorMode = cli.BoolFlag{
        Name:  "zkevm.ziskvm-emulator-mode",
        Usage: "Use ZiskVM emulator instead of full proving (for testing)",
        Value: false,
    }
    
    PreferZiskVMForFEP = cli.BoolFlag{
        Name:  "zkevm.prefer-ziskvm-for-fep",
        Usage: "Prefer ZiskVM over legacy executor for FEP generation",
        Value: true,
    }
)
```

### 4.3 Deliverables

- [ ] `zk/ziskvm_verifier/` package scaffolding
- [ ] Configuration struct extensions
- [ ] CLI flag definitions
- [ ] Interface definitions
- [ ] Unit test framework setup

---

## 5. Phase 2: ZiskVM zkEVM Program Development

### 5.1 Objectives

- [ ] Design zkEVM program architecture for ZiskVM
- [ ] Implement state transition verification in Rust
- [ ] Handle all EVM precompiles via ZiskVM syscalls
- [ ] Test with CDK-Erigon witness format

### 5.2 ZiskVM Program Structure

```
ziskvm-zkevm/
├── Cargo.toml
├── src/
│   ├── main.rs              # Entry point
│   ├── input.rs             # Input deserialization
│   ├── output.rs            # Output serialization
│   ├── state/
│   │   ├── mod.rs
│   │   ├── smt.rs           # Sparse Merkle Tree operations
│   │   ├── account.rs       # Account state handling
│   │   └── storage.rs       # Contract storage
│   ├── evm/
│   │   ├── mod.rs
│   │   ├── executor.rs      # EVM execution engine
│   │   ├── opcodes.rs       # Opcode implementations
│   │   ├── precompiles.rs   # Precompile handlers
│   │   └── gas.rs           # Gas accounting
│   ├── block/
│   │   ├── mod.rs
│   │   ├── header.rs        # Block header processing
│   │   └── transaction.rs   # Transaction processing
│   └── utils/
│       ├── mod.rs
│       ├── rlp.rs           # RLP encoding/decoding
│       └── hash.rs          # Hashing utilities
└── tests/
    ├── integration_test.rs
    └── test_vectors/
```

### 5.3 Input Format Definition

```rust
// ziskvm-zkevm/src/input.rs

use serde::{Deserialize, Serialize};

/// Main input structure matching CDK-Erigon's Payload
#[derive(Debug, Serialize, Deserialize)]
pub struct ZiskVMInput {
    /// SMT partial tree + smart contract code + old state root
    pub witness: Vec<u8>,
    
    /// Encoded datastream (transactions, block headers, batch info)
    pub data_stream: Vec<u8>,
    
    /// Sequencer address
    pub coinbase: [u8; 20],
    
    /// Previous accumulator input hash
    pub old_acc_input_hash: [u8; 32],
    
    /// L1 info tree root
    pub l1_info_root: [u8; 32],
    
    /// Timestamp limit for batch
    pub timestamp_limit: u64,
    
    /// Forced blockhash from L1 (for forced batches)
    pub forced_blockhash_l1: [u8; 32],
    
    /// Batch context identifier
    pub context_id: String,
    
    /// L1 info tree index to min timestamp mappings
    pub l1_info_tree_min_timestamps: Vec<(u64, u64)>,
    
    /// Fork ID for the batch
    pub fork_id: u64,
    
    /// Expected old state root (for verification)
    pub old_state_root: [u8; 32],
}

/// Parsed witness data
#[derive(Debug)]
pub struct ParsedWitness {
    pub accounts: Vec<AccountWitness>,
    pub storage: Vec<StorageWitness>,
    pub code: Vec<CodeWitness>,
    pub smt_proof: SMTProof,
}

/// Parsed datastream
#[derive(Debug)]
pub struct ParsedDataStream {
    pub batch_bookmark: BatchBookmark,
    pub blocks: Vec<BlockData>,
    pub ger_updates: Vec<GERUpdate>,
}

#[derive(Debug)]
pub struct BlockData {
    pub header: BlockHeader,
    pub transactions: Vec<TransactionData>,
    pub l1_info_tree_index: u64,
}
```

### 5.4 Core Execution Logic

```rust
// ziskvm-zkevm/src/main.rs

#![no_main]
#![no_std]

extern crate alloc;

use alloc::vec::Vec;
use ziskos::io::{read_input, set_output};

mod input;
mod state;
mod evm;
mod block;
mod utils;

use input::{ZiskVMInput, ParsedWitness, ParsedDataStream};
use state::StateDB;
use evm::EVMExecutor;
use block::process_block;

#[no_mangle]
pub extern "C" fn main() {
    // 1. Read and parse input
    let input: ZiskVMInput = read_input();
    
    // 2. Parse witness into usable structures
    let witness = ParsedWitness::from_bytes(&input.witness)
        .expect("Failed to parse witness");
    
    // 3. Parse datastream
    let datastream = ParsedDataStream::from_bytes(&input.data_stream)
        .expect("Failed to parse datastream");
    
    // 4. Initialize state DB from witness
    let mut state = StateDB::from_witness(&witness, input.old_state_root);
    
    // 5. Verify old state root matches
    let computed_old_root = state.compute_root();
    assert_eq!(
        computed_old_root, 
        input.old_state_root,
        "Old state root mismatch"
    );
    
    // 6. Process each block in the batch
    for block in datastream.blocks.iter() {
        let executor = EVMExecutor::new(
            &mut state,
            input.fork_id,
            input.coinbase,
            block.header.timestamp,
        );
        
        // Process block (handles GER, transactions, state updates)
        let block_result = process_block(
            executor,
            block,
            &datastream.ger_updates,
            input.l1_info_root,
            &input.l1_info_tree_min_timestamps,
        ).expect("Block execution failed");
        
        // Update state with block results
        state.apply_block_changes(&block_result);
    }
    
    // 7. Compute new state root
    let new_state_root = state.compute_root();
    
    // 8. Output results
    // Output 0: New state root (32 bytes)
    set_output(0, u64::from_le_bytes(new_state_root[0..8].try_into().unwrap()));
    set_output(1, u64::from_le_bytes(new_state_root[8..16].try_into().unwrap()));
    set_output(2, u64::from_le_bytes(new_state_root[16..24].try_into().unwrap()));
    set_output(3, u64::from_le_bytes(new_state_root[24..32].try_into().unwrap()));
    
    // Output 4: Batch number
    set_output(4, datastream.batch_bookmark.batch_number);
    
    // Output 5: Block count
    set_output(5, datastream.blocks.len() as u64);
}
```

### 5.5 Precompile Integration

```rust
// ziskvm-zkevm/src/evm/precompiles.rs

use ziskos::syscall::{
    syscall_keccak_f,
    syscall_sha256_f,
    syscall_secp256k1_add,
    syscall_secp256k1_dbl,
    syscall_bn254_curve_add,
    syscall_bn254_curve_dbl,
    syscall_arith256_mod,
};

/// EVM Precompile addresses (0x01 - 0x0a)
pub enum Precompile {
    ECRecover = 0x01,      // secp256k1 ECDSA recovery
    SHA256 = 0x02,         // SHA-256 hash
    RIPEMD160 = 0x03,      // RIPEMD-160 hash
    Identity = 0x04,       // Data copy
    ModExp = 0x05,         // Modular exponentiation
    BN254Add = 0x06,       // BN254 curve addition
    BN254Mul = 0x07,       // BN254 scalar multiplication
    BN254Pairing = 0x08,   // BN254 pairing check
    Blake2F = 0x09,        // BLAKE2 F compression
    PointEval = 0x0a,      // KZG point evaluation
}

impl Precompile {
    pub fn execute(&self, input: &[u8]) -> Result<Vec<u8>, PrecompileError> {
        match self {
            Precompile::ECRecover => ec_recover(input),
            Precompile::SHA256 => sha256_hash(input),
            Precompile::BN254Add => bn254_add(input),
            Precompile::BN254Mul => bn254_mul(input),
            // ... other precompiles
        }
    }
}

fn sha256_hash(input: &[u8]) -> Result<Vec<u8>, PrecompileError> {
    let mut state = [
        0x6a09e667u32, 0xbb67ae85u32, 0xcdf9c9bau32, 0xa54ff53au32,
        0x510e527fu32, 0x9b05688cu32, 0x1f83d9abu32, 0x5be0cd19u32,
    ];
    
    // Pad input and process blocks
    let padded = sha256_pad(input);
    for chunk in padded.chunks(64) {
        let block: [u32; 16] = chunk_to_u32_array(chunk);
        // Use ZiskVM syscall for compression function
        unsafe {
            syscall_sha256_f(state.as_mut_ptr(), block.as_ptr());
        }
    }
    
    Ok(state_to_bytes(&state))
}

fn ec_recover(input: &[u8]) -> Result<Vec<u8>, PrecompileError> {
    // Parse input: hash (32) + v (32) + r (32) + s (32)
    if input.len() != 128 {
        return Err(PrecompileError::InvalidInput);
    }
    
    let hash = &input[0..32];
    let v = input[63]; // last byte of v field
    let r = &input[64..96];
    let s = &input[96..128];
    
    // Use ZiskVM secp256k1 syscalls for recovery
    let pubkey = secp256k1_recover(hash, v, r, s)?;
    
    // Return keccak256(pubkey)[12..32] as address
    let addr_hash = keccak256(&pubkey);
    Ok(addr_hash[12..32].to_vec())
}

fn bn254_add(input: &[u8]) -> Result<Vec<u8>, PrecompileError> {
    // Parse two G1 points (64 bytes each)
    let p1 = parse_g1_point(&input[0..64])?;
    let p2 = parse_g1_point(&input[64..128])?;
    
    let mut result = [0u8; 64];
    unsafe {
        syscall_bn254_curve_add(
            result.as_mut_ptr(),
            p1.as_ptr(),
            p2.as_ptr(),
        );
    }
    
    Ok(result.to_vec())
}
```

### 5.6 Deliverables

- [ ] Complete `ziskvm-zkevm` Rust crate
- [ ] SMT implementation matching CDK-Erigon format
- [ ] EVM executor with all opcodes
- [ ] All precompiles mapped to ZiskVM syscalls
- [ ] Witness parser (compatible with Erigon format)
- [ ] DataStream parser (protobuf compatible)
- [ ] Test vectors from CDK-Erigon

---

## 6. Phase 3: CDK-Erigon Integration

### 6.1 Objectives

- [ ] Implement Go-side ZiskVM verifier
- [ ] Create witness encoder for ZiskVM format
- [ ] Build verifier multiplexer
- [ ] Integrate into sequencer stage

### 6.2 ZiskVM Verifier Implementation

```go
// zk/ziskvm_verifier/verifier.go

package ziskvm_verifier

import (
    "context"
    "fmt"
    "sync"
    "sync/atomic"
    "time"

    "github.com/erigontech/erigon-lib/common"
    "github.com/erigontech/erigon-lib/kv"
    "github.com/erigontech/erigon-lib/log/v3"
    "github.com/erigontech/erigon/eth/ethconfig"
)

type ZiskVMVerifierImpl struct {
    cfg        ethconfig.Zk
    db         kv.RwDB
    
    // Service clients for distributed proving
    services   []*ZiskVMService
    serviceIdx int
    
    // Async verification tracking
    promises    []*Promise[*VerificationResult]
    mtxPromises *sync.Mutex
    
    // Cancellation
    cancelAll  atomic.Bool
    
    // Metrics
    metrics    *VerifierMetrics
}

func NewZiskVMVerifier(
    cfg ethconfig.Zk,
    db kv.RwDB,
) (*ZiskVMVerifierImpl, error) {
    services := make([]*ZiskVMService, len(cfg.ZiskVMServiceUrls))
    for i, url := range cfg.ZiskVMServiceUrls {
        svc, err := NewZiskVMService(url, cfg.ZiskVMRequestTimeout, cfg.ZiskVMMaxConcurrentRequests)
        if err != nil {
            log.Warn("Failed to create ZiskVM service", "url", url, "err", err)
            continue
        }
        services[i] = svc
    }
    
    return &ZiskVMVerifierImpl{
        cfg:         cfg,
        db:          db,
        services:    services,
        serviceIdx:  0,
        promises:    make([]*Promise[*VerificationResult], 0),
        mtxPromises: &sync.Mutex{},
        metrics:     NewVerifierMetrics(),
    }, nil
}

func (v *ZiskVMVerifierImpl) StartAsyncVerification(
    logPrefix string,
    forkId uint64,
    batchNumber uint64,
    expectedStateRoot common.Hash,
    counters map[string]int,
    blockNumbers []uint64,
    timeout time.Duration,
    retries int,
) {
    request := &VerificationRequest{
        BatchNumber:       batchNumber,
        ForkId:            forkId,
        BlockNumbers:      blockNumbers,
        ExpectedStateRoot: expectedStateRoot,
    }
    
    promise := v.verifyAsync(request, timeout, retries)
    v.appendPromise(promise)
    
    log.Info(fmt.Sprintf("[%s] Started ZiskVM verification", logPrefix),
        "batch", batchNumber,
        "blocks", fmt.Sprintf("[%d;%d]", blockNumbers[0], blockNumbers[len(blockNumbers)-1]),
        "pending", len(v.promises),
    )
}

func (v *ZiskVMVerifierImpl) verifyAsync(
    request *VerificationRequest,
    timeout time.Duration,
    retries int,
) *Promise[*VerificationResult] {
    return NewPromise[*VerificationResult](func() (*VerificationResult, error) {
        ctx, cancel := context.WithTimeout(context.Background(), timeout)
        defer cancel()
        
        // Get witness and datastream from database
        tx, err := v.db.BeginRo(ctx)
        if err != nil {
            return nil, err
        }
        defer tx.Rollback()
        
        // Generate witness
        witness, err := v.generateWitness(ctx, tx, request)
        if err != nil {
            return nil, fmt.Errorf("witness generation failed: %w", err)
        }
        
        // Get datastream bytes
        datastream, err := v.getDataStreamBytes(ctx, tx, request)
        if err != nil {
            return nil, fmt.Errorf("datastream generation failed: %w", err)
        }
        
        // Encode for ZiskVM
        encoded, err := EncodeZiskVMInput(witness, datastream, request)
        if err != nil {
            return nil, fmt.Errorf("encoding failed: %w", err)
        }
        
        // Get available service
        svc := v.getNextAvailableService()
        if svc == nil {
            return nil, ErrNoServiceAvailable
        }
        
        // Generate and verify proof
        startTime := time.Now()
        result, err := svc.VerifyBatch(ctx, encoded, request.ExpectedStateRoot)
        if err != nil {
            return nil, fmt.Errorf("verification failed: %w", err)
        }
        
        v.metrics.RecordVerification(time.Since(startTime), result.Valid)
        
        return result, nil
    })
}

func (v *ZiskVMVerifierImpl) ProcessResultsSequentially(logPrefix string) ([]*VerificationResult, *VerificationResult) {
    v.mtxPromises.Lock()
    defer v.mtxPromises.Unlock()
    
    var results []*VerificationResult
    var failedResult *VerificationResult
    
    for idx, promise := range v.promises {
        result, err := promise.TryGet()
        if result == nil && err == nil {
            // Not ready yet
            break
        }
        
        if err != nil {
            log.Error(fmt.Sprintf("[%s] ZiskVM verification error", logPrefix), "err", err)
            // Handle retry logic...
            break
        }
        
        log.Info(fmt.Sprintf("[%s] ZiskVM verification complete", logPrefix),
            "batch", result.BatchNumber,
            "valid", result.Valid,
            "pending", len(v.promises)-1-idx,
        )
        
        if !result.Valid {
            failedResult = result
            break
        }
        
        results = append(results, result)
    }
    
    v.promises = v.promises[len(results):]
    return results, failedResult
}

func (v *ZiskVMVerifierImpl) getNextAvailableService() *ZiskVMService {
    for i := 0; i < len(v.services); i++ {
        v.serviceIdx = (v.serviceIdx + 1) % len(v.services)
        if v.services[v.serviceIdx] != nil && v.services[v.serviceIdx].CheckOnline() {
            return v.services[v.serviceIdx]
        }
    }
    return nil
}
```

### 6.3 Input Encoder

```go
// zk/ziskvm_verifier/encoder.go

package ziskvm_verifier

import (
    "bytes"
    "encoding/binary"
    "encoding/json"
    
    "github.com/erigontech/erigon-lib/common"
)

// ZiskVMInputJSON matches the Rust input structure
type ZiskVMInputJSON struct {
    Witness                   []byte            `json:"witness"`
    DataStream                []byte            `json:"data_stream"`
    Coinbase                  [20]byte          `json:"coinbase"`
    OldAccInputHash           [32]byte          `json:"old_acc_input_hash"`
    L1InfoRoot                [32]byte          `json:"l1_info_root"`
    TimestampLimit            uint64            `json:"timestamp_limit"`
    ForcedBlockhashL1         [32]byte          `json:"forced_blockhash_l1"`
    ContextId                 string            `json:"context_id"`
    L1InfoTreeMinTimestamps   [][2]uint64       `json:"l1_info_tree_min_timestamps"`
    ForkId                    uint64            `json:"fork_id"`
    OldStateRoot              [32]byte          `json:"old_state_root"`
}

func EncodeZiskVMInput(
    witness []byte,
    dataStream []byte,
    request *VerificationRequest,
) ([]byte, error) {
    // Convert L1InfoTreeMinTimestamps
    timestamps := make([][2]uint64, 0, len(request.L1InfoTreeMinTimestamps))
    for k, v := range request.L1InfoTreeMinTimestamps {
        timestamps = append(timestamps, [2]uint64{k, v})
    }
    
    input := ZiskVMInputJSON{
        Witness:                 witness,
        DataStream:              dataStream,
        Coinbase:                request.Coinbase,
        OldAccInputHash:         [32]byte{}, // TODO: populate from request
        L1InfoRoot:              copyToArray32(request.L1InfoRoot),
        TimestampLimit:          request.TimestampLimit,
        ForcedBlockhashL1:       [32]byte{},
        ContextId:               request.ContextId,
        L1InfoTreeMinTimestamps: timestamps,
        ForkId:                  request.ForkId,
        OldStateRoot:            request.OldStateRoot,
    }
    
    return json.Marshal(input)
}

// EncodeBinaryInput for more efficient binary encoding
func EncodeBinaryInput(
    witness []byte,
    dataStream []byte,
    request *VerificationRequest,
) ([]byte, error) {
    buf := new(bytes.Buffer)
    
    // Write header
    binary.Write(buf, binary.LittleEndian, uint32(len(witness)))
    binary.Write(buf, binary.LittleEndian, uint32(len(dataStream)))
    
    // Write witness
    buf.Write(witness)
    
    // Write datastream
    buf.Write(dataStream)
    
    // Write metadata
    buf.Write(request.Coinbase[:])
    buf.Write(request.OldStateRoot[:])
    buf.Write(request.L1InfoRoot)
    binary.Write(buf, binary.LittleEndian, request.TimestampLimit)
    binary.Write(buf, binary.LittleEndian, request.ForkId)
    binary.Write(buf, binary.LittleEndian, request.BatchNumber)
    
    // Write L1 info tree timestamps
    binary.Write(buf, binary.LittleEndian, uint32(len(request.L1InfoTreeMinTimestamps)))
    for k, v := range request.L1InfoTreeMinTimestamps {
        binary.Write(buf, binary.LittleEndian, k)
        binary.Write(buf, binary.LittleEndian, v)
    }
    
    return buf.Bytes(), nil
}
```

### 6.4 Verifier Multiplexer

```go
// zk/verifier_multiplexer/multiplexer.go

package verifier_multiplexer

import (
    "context"
    "time"

    "github.com/erigontech/erigon-lib/common"
    "github.com/erigontech/erigon-lib/kv"
    "github.com/erigontech/erigon/eth/ethconfig"
    legacy "github.com/erigontech/erigon/zk/legacy_executor_verifier"
    ziskvm "github.com/erigontech/erigon/zk/ziskvm_verifier"
)

type VerifierType int

const (
    VerifierTypeLegacy VerifierType = iota
    VerifierTypeZiskVM
)

// VerifierMultiplexer routes verification requests to the appropriate backend
type VerifierMultiplexer struct {
    cfg            ethconfig.Zk
    legacyVerifier *legacy.LegacyExecutorVerifier
    ziskVMVerifier *ziskvm.ZiskVMVerifierImpl
    
    // Selection strategy
    strategy       SelectionStrategy
}

type SelectionStrategy interface {
    SelectVerifier(batchNumber, forkId uint64) VerifierType
}

// DefaultStrategy prefers ZiskVM for FEP, falls back to legacy
type DefaultStrategy struct {
    cfg ethconfig.Zk
}

func (s *DefaultStrategy) SelectVerifier(batchNumber, forkId uint64) VerifierType {
    // Use legacy executor for older forks or if ZiskVM is disabled
    if !s.cfg.ZiskVMEnabled {
        return VerifierTypeLegacy
    }
    
    // Use ZiskVM for newer batches if preferred
    if s.cfg.PreferZiskVMForFEP && forkId >= s.cfg.PessimisticForkNumber {
        return VerifierTypeZiskVM
    }
    
    return VerifierTypeLegacy
}

func NewVerifierMultiplexer(
    cfg ethconfig.Zk,
    db kv.RwDB,
    legacyVerifier *legacy.LegacyExecutorVerifier,
) (*VerifierMultiplexer, error) {
    var ziskVMVerifier *ziskvm.ZiskVMVerifierImpl
    var err error
    
    if cfg.ZiskVMEnabled && cfg.HasZiskVMService() {
        ziskVMVerifier, err = ziskvm.NewZiskVMVerifier(cfg, db)
        if err != nil {
            return nil, err
        }
    }
    
    return &VerifierMultiplexer{
        cfg:            cfg,
        legacyVerifier: legacyVerifier,
        ziskVMVerifier: ziskVMVerifier,
        strategy:       &DefaultStrategy{cfg: cfg},
    }, nil
}

func (m *VerifierMultiplexer) StartAsyncVerification(
    logPrefix string,
    forkId uint64,
    batchNumber uint64,
    stateRoot common.Hash,
    counters map[string]int,
    blockNumbers []uint64,
    useRemoteExecutor bool,
    timeout time.Duration,
    retries int,
) {
    verifierType := m.strategy.SelectVerifier(batchNumber, forkId)
    
    switch verifierType {
    case VerifierTypeZiskVM:
        if m.ziskVMVerifier != nil {
            m.ziskVMVerifier.StartAsyncVerification(
                logPrefix, forkId, batchNumber, stateRoot,
                counters, blockNumbers, timeout, retries,
            )
            return
        }
        // Fall through to legacy if ZiskVM unavailable
        fallthrough
        
    case VerifierTypeLegacy:
        m.legacyVerifier.StartAsyncVerification(
            logPrefix, forkId, batchNumber, stateRoot,
            counters, blockNumbers, useRemoteExecutor, timeout, retries,
        )
    }
}

func (m *VerifierMultiplexer) HasPendingVerifications() (bool, int) {
    legacyPending, legacyCount := m.legacyVerifier.HasPendingVerifications()
    
    var ziskVMPending bool
    var ziskVMCount int
    if m.ziskVMVerifier != nil {
        ziskVMPending, ziskVMCount = m.ziskVMVerifier.HasPendingVerifications()
    }
    
    return legacyPending || ziskVMPending, legacyCount + ziskVMCount
}

func (m *VerifierMultiplexer) ProcessResultsSequentially(logPrefix string) (interface{}, interface{}) {
    // Process both verifiers and combine results
    // Implementation depends on how results should be merged
    
    legacyResults, legacyUnwind := m.legacyVerifier.ProcessResultsSequentially(logPrefix)
    
    if m.ziskVMVerifier != nil {
        ziskVMResults, ziskVMFailed := m.ziskVMVerifier.ProcessResultsSequentially(logPrefix)
        // Combine results...
        _ = ziskVMResults
        _ = ziskVMFailed
    }
    
    return legacyResults, legacyUnwind
}

func (m *VerifierMultiplexer) CancelAllRequests() {
    m.legacyVerifier.CancelAllRequests()
    if m.ziskVMVerifier != nil {
        m.ziskVMVerifier.CancelAllRequests()
    }
}
```

### 6.5 Integration into Sequencer Stage

**Modify `zk/stages/stage_sequence_execute_utils.go`**:

```go
// zk/stages/stage_sequence_execute_utils.go - MODIFICATIONS

import (
    // ... existing imports ...
    multiplexer "github.com/erigontech/erigon/zk/verifier_multiplexer"
)

type SequenceBlockCfg struct {
    // ... existing fields ...
    
    // Replace legacyVerifier with multiplexer
    verifierMultiplexer *multiplexer.VerifierMultiplexer
    
    // Keep legacy for backward compatibility
    legacyVerifier *verifier.LegacyExecutorVerifier
}

func StageSequenceBlocksCfg(
    // ... existing params ...
    legacyVerifier *verifier.LegacyExecutorVerifier,
    // NEW: optional multiplexer
    verifierMultiplexer *multiplexer.VerifierMultiplexer,
    // ... rest of params ...
) SequenceBlockCfg {
    // ... existing code ...
    
    cfg := SequenceBlockCfg{
        // ... existing fields ...
        legacyVerifier:      legacyVerifier,
        verifierMultiplexer: verifierMultiplexer,
    }
    
    return cfg
}

// Helper to get the active verifier
func (cfg *SequenceBlockCfg) GetVerifier() interface{} {
    if cfg.verifierMultiplexer != nil {
        return cfg.verifierMultiplexer
    }
    return cfg.legacyVerifier
}
```

**Modify `zk/stages/stage_sequence_execute.go`**:

```go
// zk/stages/stage_sequence_execute.go - LINE 808 MODIFICATION

// BEFORE:
// cfg.legacyVerifier.StartAsyncVerification(...)

// AFTER:
if cfg.verifierMultiplexer != nil {
    cfg.verifierMultiplexer.StartAsyncVerification(
        batchContext.s.LogPrefix(),
        batchState.forkId,
        batchState.batchNumber,
        block.Root(),
        counters.UsedAsMap(),
        batchState.builtBlocks,
        useExecutorForVerification,
        batchContext.cfg.zk.SequencerBatchVerificationTimeout,
        batchContext.cfg.zk.SequencerBatchVerificationRetries,
    )
} else {
    // Fallback to legacy verifier
    cfg.legacyVerifier.StartAsyncVerification(
        batchContext.s.LogPrefix(),
        batchState.forkId,
        batchState.batchNumber,
        block.Root(),
        counters.UsedAsMap(),
        batchState.builtBlocks,
        useExecutorForVerification,
        batchContext.cfg.zk.SequencerBatchVerificationTimeout,
        batchContext.cfg.zk.SequencerBatchVerificationRetries,
    )
}
```

### 6.6 Deliverables

- [ ] `zk/ziskvm_verifier/verifier.go` - Main verifier
- [ ] `zk/ziskvm_verifier/encoder.go` - Input encoding
- [ ] `zk/ziskvm_verifier/service.go` - gRPC/HTTP client
- [ ] `zk/verifier_multiplexer/multiplexer.go` - Router
- [ ] Modified `stage_sequence_execute.go`
- [ ] Modified `stage_sequence_execute_utils.go`
- [ ] CLI flag integration in `flags_zkevm.go`
- [ ] Integration tests

---

## 7. Phase 4: Verification & AggLayer Integration

### 7.1 Objectives

- [ ] Implement proof verification on Go side
- [ ] Create proof storage mechanism
- [ ] Design AggLayer submission flow
- [ ] Handle dual-proof scenarios (FEP + PP)

### 7.2 Proof Storage

```go
// zk/ziskvm_verifier/proof_store.go

package ziskvm_verifier

import (
    "encoding/json"
    "os"
    "path"
    "sync"
    
    "github.com/erigontech/erigon-lib/kv"
)

type ProofStore struct {
    db        kv.RwDB
    outputDir string
    mtx       sync.RWMutex
}

func NewProofStore(db kv.RwDB, outputDir string) *ProofStore {
    if outputDir != "" {
        os.MkdirAll(outputDir, 0755)
    }
    return &ProofStore{db: db, outputDir: outputDir}
}

func (s *ProofStore) StoreProof(batchNumber uint64, proof *ZiskProof) error {
    s.mtx.Lock()
    defer s.mtx.Unlock()
    
    // Store to database
    tx, err := s.db.BeginRw(context.Background())
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    key := make([]byte, 8)
    binary.BigEndian.PutUint64(key, batchNumber)
    
    value, err := json.Marshal(proof)
    if err != nil {
        return err
    }
    
    if err := tx.Put(kv.TableZiskVMProofs, key, value); err != nil {
        return err
    }
    
    // Also write to file if output directory configured
    if s.outputDir != "" {
        filename := path.Join(s.outputDir, fmt.Sprintf("proof_%d.json", batchNumber))
        if err := os.WriteFile(filename, value, 0644); err != nil {
            log.Warn("Failed to write proof to file", "err", err)
        }
    }
    
    return tx.Commit()
}

func (s *ProofStore) GetProof(batchNumber uint64) (*ZiskProof, error) {
    s.mtx.RLock()
    defer s.mtx.RUnlock()
    
    tx, err := s.db.BeginRo(context.Background())
    if err != nil {
        return nil, err
    }
    defer tx.Rollback()
    
    key := make([]byte, 8)
    binary.BigEndian.PutUint64(key, batchNumber)
    
    value, err := tx.GetOne(kv.TableZiskVMProofs, key)
    if err != nil {
        return nil, err
    }
    if value == nil {
        return nil, nil
    }
    
    var proof ZiskProof
    if err := json.Unmarshal(value, &proof); err != nil {
        return nil, err
    }
    
    return &proof, nil
}
```

### 7.3 AggLayer Proof Submission

```go
// zk/ziskvm_verifier/agglayer.go

package ziskvm_verifier

import (
    "context"
    "encoding/json"
    "net/http"
    "bytes"
)

// AggLayerClient handles proof submission to AggLayer
type AggLayerClient struct {
    endpoint string
    chainId  uint64
    client   *http.Client
}

type AggLayerProofSubmission struct {
    ChainId       uint64   `json:"chain_id"`
    BatchNumber   uint64   `json:"batch_number"`
    ProofType     string   `json:"proof_type"` // "fep" or "pp"
    Proof         []byte   `json:"proof"`
    PublicInputs  []byte   `json:"public_inputs"`
    NewStateRoot  [32]byte `json:"new_state_root"`
    OldStateRoot  [32]byte `json:"old_state_root"`
    BlockRange    [2]uint64 `json:"block_range"`
}

func NewAggLayerClient(endpoint string, chainId uint64) *AggLayerClient {
    return &AggLayerClient{
        endpoint: endpoint,
        chainId:  chainId,
        client:   &http.Client{Timeout: 30 * time.Second},
    }
}

func (c *AggLayerClient) SubmitFEPProof(ctx context.Context, proof *ZiskProof, result *VerificationResult) error {
    submission := AggLayerProofSubmission{
        ChainId:      c.chainId,
        BatchNumber:  proof.BatchNumber,
        ProofType:    "fep",
        Proof:        proof.ProofBytes,
        PublicInputs: proof.PublicInputs,
        NewStateRoot: result.NewStateRoot,
        // ... fill other fields
    }
    
    body, err := json.Marshal(submission)
    if err != nil {
        return err
    }
    
    req, err := http.NewRequestWithContext(ctx, "POST", c.endpoint+"/submit_proof", bytes.NewReader(body))
    if err != nil {
        return err
    }
    req.Header.Set("Content-Type", "application/json")
    
    resp, err := c.client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("AggLayer submission failed: %d", resp.StatusCode)
    }
    
    return nil
}
```

### 7.4 Deliverables

- [ ] `zk/ziskvm_verifier/proof_store.go` - Proof persistence
- [ ] `zk/ziskvm_verifier/agglayer.go` - AggLayer client
- [ ] Database schema for ZiskVM proofs
- [ ] Proof verification tests

---

## 8. Phase 5: Optimization & Production Readiness

### 8.1 Objectives

- [ ] Performance optimization
- [ ] GPU acceleration integration
- [ ] MPI distributed proving
- [ ] Monitoring and metrics
- [ ] Production deployment guide

### 8.2 GPU Acceleration

```go
// zk/ziskvm_verifier/service.go - GPU Configuration

type ZiskVMService struct {
    url             string
    timeout         time.Duration
    maxConcurrent   int
    useGPU          bool
    gpuDeviceId     int
    conn            *grpc.ClientConn
    client          ZiskVMServiceClient
    semaphore       chan struct{}
}

type ProveRequest struct {
    Input          []byte `json:"input"`
    ProgramPath    string `json:"program_path"`
    UseGPU         bool   `json:"use_gpu"`
    GPUDeviceId    int    `json:"gpu_device_id,omitempty"`
    MPIWorldSize   int    `json:"mpi_world_size,omitempty"`
}
```

### 8.3 Metrics

```go
// zk/ziskvm_verifier/metrics.go

package ziskvm_verifier

import (
    "sync"
    "time"
    
    "github.com/prometheus/client_golang/prometheus"
)

var (
    ziskVMProvingDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "ziskvm_proving_duration_seconds",
            Help:    "Time spent generating ZiskVM proofs",
            Buckets: []float64{1, 5, 10, 30, 60, 120, 300, 600},
        },
        []string{"batch_size", "success"},
    )
    
    ziskVMVerificationCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "ziskvm_verifications_total",
            Help: "Total number of ZiskVM verifications",
        },
        []string{"result"},
    )
    
    ziskVMWitnessSize = prometheus.NewHistogram(
        prometheus.HistogramOpts{
            Name:    "ziskvm_witness_size_bytes",
            Help:    "Size of witness data in bytes",
            Buckets: prometheus.ExponentialBuckets(1024, 2, 20),
        },
    )
    
    ziskVMProofSize = prometheus.NewHistogram(
        prometheus.HistogramOpts{
            Name:    "ziskvm_proof_size_bytes",
            Help:    "Size of generated proofs in bytes",
            Buckets: prometheus.ExponentialBuckets(1024, 2, 15),
        },
    )
)

func init() {
    prometheus.MustRegister(
        ziskVMProvingDuration,
        ziskVMVerificationCounter,
        ziskVMWitnessSize,
        ziskVMProofSize,
    )
}

type VerifierMetrics struct {
    mtx                sync.Mutex
    totalVerifications int64
    successfulProofs   int64
    failedProofs       int64
    totalProvingTime   time.Duration
}

func (m *VerifierMetrics) RecordVerification(duration time.Duration, success bool) {
    m.mtx.Lock()
    defer m.mtx.Unlock()
    
    m.totalVerifications++
    m.totalProvingTime += duration
    
    result := "success"
    if success {
        m.successfulProofs++
    } else {
        m.failedProofs++
        result = "failure"
    }
    
    ziskVMVerificationCounter.WithLabelValues(result).Inc()
    ziskVMProvingDuration.WithLabelValues("default", result).Observe(duration.Seconds())
}
```

### 8.4 Production Configuration Example

```yaml
# hermezconfig-ziskvm.yaml

zkevm:
  # Existing executor (for fallback)
  executor-urls: "executor-legacy:50071"
  executor-enabled: true
  executor-strict: false
  
  # ZiskVM configuration
  ziskvm-enabled: true
  ziskvm-service-urls: "ziskvm-prover-1:50081,ziskvm-prover-2:50081"
  ziskvm-program-path: "/app/ziskvm/zkevm.elf"
  ziskvm-proving-key-path: "/app/ziskvm/proving.key"
  ziskvm-verification-key-path: "/app/ziskvm/verification.key"
  ziskvm-max-concurrent-requests: 4
  ziskvm-request-timeout: "10m"
  ziskvm-proof-output-path: "/data/proofs"
  ziskvm-use-gpu: true
  ziskvm-mpi-enabled: true
  ziskvm-emulator-mode: false
  
  # Proof strategy
  prefer-ziskvm-for-fep: true
  fallback-to-legacy-on-ziskvm-failure: true
  
  # Pessimistic proofs (unchanged)
  pessimistic-fork-number: 12
```

### 8.5 Deliverables

- [ ] `zk/ziskvm_verifier/metrics.go` - Prometheus metrics
- [ ] GPU-enabled service configuration
- [ ] MPI distributed proving setup
- [ ] Production deployment documentation
- [ ] Performance benchmarks

---

## 9. File Modification Matrix

### 9.1 Files to Modify

| File | Change Type | Description |
|------|-------------|-------------|
| `eth/ethconfig/config_zkevm.go` | Extend | Add ZiskVM configuration fields |
| `cmd/utils/flags.go` | Extend | Add ZiskVM CLI flags |
| `turbo/cli/flags_zkevm.go` | Extend | Apply ZiskVM flags to config |
| `turbo/cli/default_flags.go` | Extend | Register ZiskVM flags |
| `zk/stages/stage_sequence_execute_utils.go` | Modify | Add verifier multiplexer support |
| `zk/stages/stage_sequence_execute.go` | Modify | Use multiplexer for verification |
| `zk/stages/stage_sequence_execute_batch.go` | Modify | Update batch stream writer |
| `node/node.go` | Modify | Initialize ZiskVM verifier |
| `erigon-lib/kv/tables.go` | Extend | Add ZiskVM proof table |
| `turbo/app/make_app.go` | Modify | Wire ZiskVM components |

### 9.2 Files to Create

| File | Purpose |
|------|---------|
| `zk/ziskvm_verifier/verifier.go` | Main verifier implementation |
| `zk/ziskvm_verifier/encoder.go` | Input encoding |
| `zk/ziskvm_verifier/decoder.go` | Output decoding |
| `zk/ziskvm_verifier/service.go` | gRPC/HTTP client |
| `zk/ziskvm_verifier/config.go` | Configuration handling |
| `zk/ziskvm_verifier/proof.go` | Proof structures |
| `zk/ziskvm_verifier/proof_store.go` | Proof persistence |
| `zk/ziskvm_verifier/agglayer.go` | AggLayer integration |
| `zk/ziskvm_verifier/metrics.go` | Prometheus metrics |
| `zk/ziskvm_verifier/interfaces.go` | Interface definitions |
| `zk/ziskvm_verifier/errors.go` | Error definitions |
| `zk/ziskvm_verifier/promise.go` | Async promise implementation |
| `zk/ziskvm_verifier/proto/ziskvm_service.proto` | gRPC definitions |
| `zk/verifier_multiplexer/multiplexer.go` | Verifier router |
| `zk/verifier_multiplexer/strategy.go` | Selection strategies |

---

## 10. New Components to Create

### 10.1 ZiskVM zkEVM Program (Rust)

```
ziskvm-zkevm/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── input.rs
│   ├── output.rs
│   ├── state/
│   │   ├── mod.rs
│   │   ├── smt.rs
│   │   ├── account.rs
│   │   └── storage.rs
│   ├── evm/
│   │   ├── mod.rs
│   │   ├── executor.rs
│   │   ├── opcodes.rs
│   │   ├── precompiles.rs
│   │   └── gas.rs
│   ├── block/
│   │   ├── mod.rs
│   │   ├── header.rs
│   │   └── transaction.rs
│   └── utils/
│       ├── mod.rs
│       ├── rlp.rs
│       └── hash.rs
└── tests/
```

### 10.2 ZiskVM Prover Service

```
ziskvm-prover-service/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── server.rs
│   ├── prover.rs
│   └── config.rs
├── proto/
│   └── prover.proto
└── Dockerfile
```

---

## 11. Testing Strategy

### 11.1 Unit Tests

| Component | Test Focus |
|-----------|------------|
| Encoder | Input serialization correctness |
| Decoder | Output parsing accuracy |
| Service | Connection handling, error recovery |
| Multiplexer | Routing logic, fallback behavior |
| ProofStore | Persistence, retrieval |

### 11.2 Integration Tests

1. **Witness Compatibility Test**
   - Generate witness from Erigon
   - Parse in ZiskVM program
   - Verify SMT root computation matches

2. **Full Batch Verification Test**
   - Execute batch in Erigon
   - Generate ZiskVM proof
   - Verify proof validity
   - Compare state roots

3. **Failover Test**
   - Simulate ZiskVM service failure
   - Verify fallback to legacy executor
   - Check no data loss

### 11.3 Performance Tests

1. **Latency Benchmarks**
   - Measure proving time per batch size
   - Compare with legacy executor

2. **Throughput Tests**
   - Concurrent proof generation
   - GPU vs CPU performance

3. **Resource Usage**
   - Memory consumption
   - CPU/GPU utilization

### 11.4 Test Vectors

Create test vectors from:
- Mainnet historical batches
- Testnet batches
- Synthetic edge cases (large transactions, complex contracts)

---

## 12. Risks and Mitigations

### 12.1 Technical Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Witness format incompatibility | High | Extensive format validation, versioning |
| ZiskVM precompile performance | Medium | Benchmark early, optimize syscalls |
| State root mismatch | High | Comprehensive test vectors, fuzzing |
| Memory constraints in ZiskVM | Medium | Witness size limits, chunked processing |

### 12.2 Operational Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| ZiskVM service unavailability | High | Automatic fallback to legacy executor |
| Proof generation latency | Medium | GPU acceleration, parallel proving |
| Configuration complexity | Low | Sensible defaults, documentation |

### 12.3 Security Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Soundness vulnerability in ZiskVM | Critical | Use audited ZiskVM version, verify proofs |
| Witness manipulation | High | Cryptographic witness commitment |
| Service impersonation | Medium | mTLS, authentication |

---

## 13. Timeline Estimates

### 13.1 Phase Breakdown

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 1: Foundation | 2-3 weeks | None |
| Phase 2: zkEVM Program | 6-8 weeks | Phase 1 |
| Phase 3: CDK-Erigon Integration | 3-4 weeks | Phase 1, 2 |
| Phase 4: Verification & AggLayer | 2-3 weeks | Phase 3 |
| Phase 5: Optimization | 3-4 weeks | Phase 4 |

### 13.2 Total Estimated Timeline

**Minimum**: 16 weeks (4 months)
**Realistic**: 20-24 weeks (5-6 months)
**Conservative**: 28 weeks (7 months)

### 13.3 Parallelization Opportunities

- Phase 2 (Rust) can proceed in parallel with Phase 1 (Go) after interfaces defined
- Testing can begin during Phase 3
- Documentation throughout all phases

---

## 14. Appendices

### 14.1 Glossary

| Term | Definition |
|------|------------|
| FEP | Full Execution Proof - proves entire batch execution |
| PP | Pessimistic Proof - proves bridge state safety |
| SMT | Sparse Merkle Tree - efficient state storage |
| AggLayer | Polygon's aggregation layer for cross-chain security |
| ZiskVM | Zero-knowledge Virtual Machine (RISC-V based) |
| DataStream | Encoded batch data (transactions, headers, etc.) |
| Witness | State data needed for proof generation |

### 14.2 References

- [ZiskVM Documentation](https://0xpolygonhermez.github.io/zisk)
- [CDK-Erigon Architecture](https://docs.agglayer.dev/cdk/cdk-erigon/architecture/)
- [Pessimistic Proofs](https://docs.polygon.technology/cdk/concepts/pessimistic-proofs/)
- [AggLayer](https://www.agglayer.dev/)
- [Plonky3](https://github.com/Plonky3/Plonky3)

### 14.3 Current Codebase Key Files

```
zk/
├── legacy_executor_verifier/
│   ├── legacy_executor_verifier.go  # Main verifier (lines 1-508)
│   ├── executor.go                   # gRPC client (lines 1-324)
│   ├── promise.go                    # Async handling
│   └── proto/
│       └── process_batch.proto       # gRPC definitions
├── witness/
│   └── witness.go                    # Witness generator (lines 1-331)
├── stages/
│   ├── stage_sequence_execute.go     # Main sequencer (lines 1-886)
│   └── stage_sequence_execute_utils.go # Utilities (lines 1-700)
└── datastream/
    └── server/
        └── data_stream_server.go     # DataStream handling
```

---

## Conclusion

This integration plan provides a comprehensive roadmap for incorporating ZiskVM-based Full Execution Proofs into CDK-Erigon while maintaining the existing SP1-based Pessimistic Proof system. The dual-proof architecture ensures:

1. **AggLayer Security**: Pessimistic proofs (SP1) continue to protect bridge operations
2. **Execution Verification**: ZiskVM provides efficient FEP generation
3. **Backward Compatibility**: Legacy executor remains available as fallback
4. **Flexibility**: Configuration-driven selection between proof systems
5. **Scalability**: GPU/MPI acceleration for production workloads

The phased approach minimizes risk while enabling incremental delivery and testing.

---

*Document Version: 1.0*
*Last Updated: November 26, 2025*
*Author: AI Assistant*
