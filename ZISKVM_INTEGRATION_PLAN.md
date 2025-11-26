# ZisKVM Integration Plan for Erigon FEP Proving

## Executive Summary

This document outlines a comprehensive plan to integrate **ZisKVM** (Zero-Knowledge Virtual Machine) into Erigon to replace the existing slow and expensive Full Execution Proof (FEP) proving system. ZisKVM is a high-performance zkVM built on Plonky3 that enables efficient zero-knowledge proof generation for arbitrary program execution.

## Current Architecture Analysis

### Existing FEP Proving System

1. **Witness Generation** (`zk/witness/witness.go`)
   - Generates SMT (Sparse Merkle Tree) witnesses from block execution state
   - Supports witness caching for performance
   - Creates witnesses for batch ranges or specific batches
   - Uses `BuildWitnessFromTrieDbState` to construct witnesses from state trie

2. **DataStream Generation** (`zk/datastream/`)
   - Creates protobuf-encoded data streams containing:
     - Transactions
     - Block headers
     - L1 info tree updates
     - Global exit roots
     - Batch bookmarks
   - Used as input to the executor/prover

3. **Legacy Executor Verifier** (`zk/legacy_executor_verifier/`)
   - Communicates with external executor via gRPC
   - Sends witness + datastream to executor
   - Receives execution results and state roots
   - Validates state root matches
   - Currently uses `ProcessStatelessBatchV2` gRPC endpoint

4. **RPC Endpoints** (`turbo/jsonrpc/zkevm_api.go`)
   - `zkevm_getBatchWitness` - Generates witness for a batch
   - `zkevm_getProverInput` - Gets prover input (witness + datastream)
   - Supports witness caching for performance

5. **Current Flow**
   ```
   Block Execution → Witness Generation → DataStream Creation → 
   Executor Verification (gRPC) → State Root Validation
   ```

### Key Components to Replace/Enhance

- **Executor Interface**: Currently uses gRPC to legacy executor
- **Proof Generation**: Currently handled by external executor
- **Witness Format**: SMT-based witness format
- **DataStream Format**: Protobuf-encoded batch data

## ZisKVM Overview

### What is ZisKVM?

- **High-performance zkVM** built on Plonky3
- **Rust-based** execution environment (RISC-V architecture)
- **Multiple interfaces**: JSON-RPC, gRPC, CLI
- **Flexible integration**: Standalone service or library
- **Optimized proof generation** with low latency

### Key Features

- No recompilation required across different programs
- Standardized prover interface
- Decentralized architecture for trustless proof generation
- Fully open-source (Polygon zkEVM + Plonky3)

### ZisKVM Architecture

ZisKVM operates as:
1. **zkVM Runtime**: Executes programs in RISC-V environment
2. **Proof Generator**: Generates ZK proofs of execution
3. **Prover Service**: Can run as standalone service or embedded

## Integration Strategy

### Phase 1: Research & Design (Week 1-2)

#### 1.1 ZisKVM Deep Dive
- [ ] Review ZisKVM source code and architecture
- [ ] Understand ZisKVM's input/output formats
- [ ] Study ZisKVM's proof generation API (JSON-RPC/gRPC)
- [ ] Analyze performance characteristics and requirements
- [ ] Review ZisKVM's witness/proof data structures

#### 1.2 Interface Design
- [ ] Design new `ZiskvmProver` interface to replace `LegacyExecutorVerifier`
- [ ] Define witness format compatibility layer (if needed)
- [ ] Design proof result structures
- [ ] Plan migration path for existing code

#### 1.3 Compatibility Analysis
- [ ] Map current witness format to ZisKVM input format
- [ ] Map current datastream to ZisKVM program input
- [ ] Identify any format conversions needed
- [ ] Plan backward compatibility strategy

### Phase 2: Core Integration (Week 3-5)

#### 2.1 Create ZisKVM Prover Package

**Location**: `zk/ziskvm_prover/`

**Structure**:
```
zk/ziskvm_prover/
├── client.go          # ZisKVM client (JSON-RPC/gRPC)
├── prover.go          # Main prover interface implementation
├── types.go           # ZisKVM-specific types
├── witness_adapter.go # Witness format adapter
├── proof_validator.go # Proof validation logic
└── config.go          # Configuration
```

**Key Components**:

1. **ZiskvmProver Interface** (`prover.go`)
   ```go
   type ZiskvmProver interface {
       GenerateProof(ctx context.Context, input *ProverInput) (*ProofResult, error)
       VerifyProof(ctx context.Context, proof *Proof) (bool, error)
       GetProofStatus(ctx context.Context, proofId string) (*ProofStatus, error)
   }
   ```

2. **ZisKVM Client** (`client.go`)
   - JSON-RPC client for ZisKVM service
   - gRPC client (if ZisKVM supports it)
   - Connection pooling and retry logic
   - Health checks

3. **Witness Adapter** (`witness_adapter.go`)
   - Converts Erigon witness format to ZisKVM input format
   - Handles format transformations
   - Validates witness compatibility

#### 2.2 Implement Witness Format Adapter

**Challenge**: ZisKVM may use different witness format than current SMT-based witness.

**Solution**:
- Create adapter layer that converts current witness format to ZisKVM format
- Or: Modify witness generation to produce ZisKVM-compatible format directly
- Support both formats during transition period

**Implementation**:
```go
type WitnessAdapter struct {
    // Conversion logic
}

func (a *WitnessAdapter) ConvertToZiskvmFormat(erigonWitness []byte) (*ZiskvmInput, error) {
    // Convert SMT witness to ZisKVM input format
}
```

#### 2.3 Integrate with DataStream

**Current**: DataStream is protobuf-encoded batch data

**Integration**:
- Use DataStream as program input to ZisKVM
- May need to convert protobuf to ZisKVM's expected format
- Ensure all batch metadata is preserved

#### 2.4 Replace Legacy Executor Calls

**Locations to Update**:
1. `zk/legacy_executor_verifier/legacy_executor_verifier.go`
   - Replace `VerifyAsync` to use ZisKVM
   - Replace `VerifySync` to use ZisKVM

2. `zk/stages/stage_sequence_execute.go`
   - Update verification calls to use ZisKVM prover

3. `turbo/jsonrpc/zkevm_api.go`
   - Update `GetProverInput` to support ZisKVM format

### Phase 3: Proof Generation Integration (Week 6-7)

#### 3.1 Implement Proof Generation Flow

**New Flow**:
```
Block Execution → Witness Generation → DataStream Creation → 
ZisKVM Proof Generation → Proof Validation → State Root Verification
```

**Implementation Steps**:

1. **Create ZisKVM Prover Service Wrapper**
   ```go
   type ZiskvmProverService struct {
       client     *ZiskvmClient
       config     *ZiskvmConfig
       witnessGen WitnessGenerator
   }
   
   func (s *ZiskvmProverService) GenerateProofForBatch(
       ctx context.Context,
       batchNum uint64,
       witness []byte,
       datastream []byte,
   ) (*ProofResult, error) {
       // 1. Prepare ZisKVM input
       input := s.prepareInput(witness, datastream)
       
       // 2. Call ZisKVM proof generation
       proof, err := s.client.GenerateProof(ctx, input)
       
       // 3. Validate proof
       return s.validateProof(proof)
   }
   ```

2. **Integrate with Stage Execution**
   - Modify `stage_sequence_execute.go` to use ZisKVM prover
   - Replace executor verification calls
   - Update error handling

3. **Update RPC Endpoints**
   - Modify `zkevm_getProverInput` to return ZisKVM-compatible input
   - Add new endpoint `zkevm_generateProof` (optional)
   - Update `zkevm_getBatchWitness` if needed

#### 3.2 Proof Validation

**Validation Steps**:
1. Verify proof structure
2. Verify proof correctness (using ZisKVM verifier)
3. Verify state root matches expected
4. Verify batch counters match

**Implementation**:
```go
func (s *ZiskvmProverService) ValidateProof(
    ctx context.Context,
    proof *Proof,
    expectedStateRoot common.Hash,
) (bool, error) {
    // 1. Verify proof with ZisKVM
    valid, err := s.client.VerifyProof(ctx, proof)
    if err != nil || !valid {
        return false, err
    }
    
    // 2. Verify state root
    if proof.StateRoot != expectedStateRoot {
        return false, ErrStateRootMismatch
    }
    
    return true, nil
}
```

### Phase 4: Configuration & Deployment (Week 8)

#### 4.1 Configuration Management

**New Configuration Flags** (`eth/ethconfig/config_zkevm.go`):
```go
type Zk struct {
    // ... existing fields ...
    
    // ZisKVM Configuration
    ZiskvmEnabled              bool
    ZiskvmServiceUrl           string        // ZisKVM service URL (JSON-RPC/gRPC)
    ZiskvmTimeout              time.Duration
    ZiskvmMaxConcurrentProofs int
    ZiskvmProofCacheEnabled    bool
    ZiskvmWitnessFormat        string        // "legacy" or "ziskvm"
}
```

**CLI Flags** (`cmd/utils/flags.go`):
```go
ZiskvmEnabledFlag = cli.BoolFlag{
    Name:  "zkevm.ziskvm-enabled",
    Usage: "Enable ZisKVM for proof generation (replaces legacy executor)",
}

ZiskvmServiceUrlFlag = cli.StringFlag{
    Name:  "zkevm.ziskvm-service-url",
    Usage: "ZisKVM service URL (JSON-RPC or gRPC endpoint)",
    Value: "http://localhost:8080",
}
```

#### 4.2 Service Discovery & Health Checks

**Implementation**:
- Health check endpoint for ZisKVM service
- Automatic reconnection on failure
- Load balancing if multiple ZisKVM instances

#### 4.3 Migration Strategy

**Gradual Migration**:
1. **Phase 1**: Run both systems in parallel, compare results
2. **Phase 2**: Route new batches to ZisKVM, keep legacy for old batches
3. **Phase 3**: Fully migrate to ZisKVM
4. **Phase 4**: Remove legacy executor code (optional)

**Feature Flag**:
```go
if cfg.Zk.ZiskvmEnabled {
    prover = NewZiskvmProver(cfg)
} else {
    prover = NewLegacyExecutorVerifier(cfg)
}
```

### Phase 5: Testing & Optimization (Week 9-10)

#### 5.1 Unit Tests

**Test Coverage**:
- [ ] Witness format conversion
- [ ] Proof generation
- [ ] Proof validation
- [ ] Error handling
- [ ] Retry logic
- [ ] Connection management

#### 5.2 Integration Tests

**Test Scenarios**:
- [ ] Single batch proof generation
- [ ] Multiple batch proof generation
- [ ] Proof validation with correct state root
- [ ] Proof validation with incorrect state root (should fail)
- [ ] Service failure and recovery
- [ ] Concurrent proof generation

#### 5.3 Performance Testing

**Metrics to Measure**:
- Proof generation time vs legacy executor
- Memory usage
- CPU usage
- Network bandwidth
- Throughput (proofs per second)

**Benchmarks**:
- Compare ZisKVM vs legacy executor
- Measure witness generation time
- Measure proof generation time
- Measure end-to-end latency

#### 5.4 Load Testing

**Scenarios**:
- High batch throughput
- Large batch sizes
- Concurrent proof requests
- Service failure scenarios

### Phase 6: Documentation & Cleanup (Week 11-12)

#### 6.1 Documentation

**Documents to Create/Update**:
- [ ] ZisKVM integration guide
- [ ] Configuration reference
- [ ] Migration guide from legacy executor
- [ ] API documentation updates
- [ ] Performance benchmarks

#### 6.2 Code Cleanup

**Tasks**:
- [ ] Remove unused legacy executor code (if fully migrated)
- [ ] Refactor common code
- [ ] Update comments and documentation
- [ ] Code review and optimization

## Technical Implementation Details

### 1. ZisKVM Client Implementation

**JSON-RPC Client** (`zk/ziskvm_prover/client.go`):
```go
type ZiskvmClient struct {
    rpcClient *rpc.Client
    url       string
    timeout   time.Duration
}

func (c *ZiskvmClient) GenerateProof(ctx context.Context, input *ProverInput) (*Proof, error) {
    var proof Proof
    err := c.rpcClient.CallContext(ctx, &proof, "ziskvm_generateProof", input)
    return &proof, err
}

func (c *ZiskvmClient) VerifyProof(ctx context.Context, proof *Proof) (bool, error) {
    var result bool
    err := c.rpcClient.CallContext(ctx, &result, "ziskvm_verifyProof", proof)
    return result, err
}
```

### 2. Witness Format Conversion

**Current Format**: SMT-based witness (trie operations)

**ZisKVM Format**: TBD (needs research)

**Adapter Implementation**:
```go
type WitnessConverter struct {
    // Conversion logic
}

func (c *WitnessConverter) Convert(witness []byte) (*ZiskvmWitness, error) {
    // Parse Erigon witness
    erigonWitness, err := witness.ParseWitnessFromBytes(witness)
    if err != nil {
        return nil, err
    }
    
    // Convert to ZisKVM format
    ziskvmWitness := c.convertToZiskvmFormat(erigonWitness)
    
    return ziskvmWitness, nil
}
```

### 3. Proof Result Structure

```go
type ProofResult struct {
    Proof      []byte          // ZK proof bytes
    StateRoot  common.Hash     // Final state root
    Counters   map[string]int  // Execution counters
    BatchNum   uint64          // Batch number
    BlockNums  []uint64        // Block numbers in batch
    Valid      bool            // Proof validity
    Error      error           // Error if proof generation failed
}
```

### 4. Error Handling

**Error Types**:
```go
var (
    ErrZiskvmServiceUnavailable = errors.New("ZisKVM service unavailable")
    ErrProofGenerationFailed    = errors.New("proof generation failed")
    ErrProofValidationFailed    = errors.New("proof validation failed")
    ErrWitnessFormatInvalid      = errors.New("invalid witness format")
    ErrStateRootMismatch         = errors.New("state root mismatch")
)
```

### 5. Retry Logic

**Implementation**:
```go
func (c *ZiskvmClient) GenerateProofWithRetry(
    ctx context.Context,
    input *ProverInput,
    maxRetries int,
) (*Proof, error) {
    var lastErr error
    for i := 0; i < maxRetries; i++ {
        proof, err := c.GenerateProof(ctx, input)
        if err == nil {
            return proof, nil
        }
        lastErr = err
        time.Sleep(time.Duration(i+1) * time.Second) // Exponential backoff
    }
    return nil, fmt.Errorf("proof generation failed after %d retries: %w", maxRetries, lastErr)
}
```

## Migration Path

### Step 1: Parallel Operation
- Deploy ZisKVM service alongside legacy executor
- Route test batches to ZisKVM
- Compare results with legacy executor
- Fix any discrepancies

### Step 2: Gradual Rollout
- Enable ZisKVM for new batches
- Keep legacy executor for old batches
- Monitor performance and correctness

### Step 3: Full Migration
- Route all batches to ZisKVM
- Keep legacy executor as fallback
- Monitor for issues

### Step 4: Legacy Removal (Optional)
- Remove legacy executor code
- Clean up unused dependencies
- Update documentation

## Risk Mitigation

### Risks

1. **Format Incompatibility**
   - **Risk**: ZisKVM may not support current witness format
   - **Mitigation**: Create adapter layer, support both formats

2. **Performance Regression**
   - **Risk**: ZisKVM may be slower than legacy executor
   - **Mitigation**: Extensive benchmarking, optimization

3. **Service Availability**
   - **Risk**: ZisKVM service may be unavailable
   - **Mitigation**: Health checks, retry logic, fallback to legacy

4. **Proof Correctness**
   - **Risk**: ZisKVM proofs may not match legacy executor
   - **Mitigation**: Parallel validation, extensive testing

5. **Integration Complexity**
   - **Risk**: Integration may be more complex than expected
   - **Mitigation**: Phased approach, thorough testing

## Success Criteria

1. **Functionality**
   - [ ] All batches can be proven using ZisKVM
   - [ ] Proofs are validated correctly
   - [ ] State roots match expected values

2. **Performance**
   - [ ] Proof generation time ≤ legacy executor (or better)
   - [ ] Memory usage is reasonable
   - [ ] Throughput meets requirements

3. **Reliability**
   - [ ] Service handles failures gracefully
   - [ ] Retry logic works correctly
   - [ ] No data loss or corruption

4. **Maintainability**
   - [ ] Code is well-documented
   - [ ] Tests have good coverage
   - [ ] Easy to configure and deploy

## Timeline Summary

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| Phase 1: Research & Design | 2 weeks | Design document, interface definitions |
| Phase 2: Core Integration | 3 weeks | ZisKVM prover package, witness adapter |
| Phase 3: Proof Generation | 2 weeks | Proof generation integration |
| Phase 4: Configuration | 1 week | Configuration flags, deployment setup |
| Phase 5: Testing | 2 weeks | Tests, benchmarks, performance analysis |
| Phase 6: Documentation | 2 weeks | Documentation, code cleanup |
| **Total** | **12 weeks** | **Full ZisKVM integration** |

## Next Steps

1. **Immediate Actions**:
   - [ ] Review ZisKVM source code and documentation
   - [ ] Set up ZisKVM development environment
   - [ ] Create proof-of-concept integration
   - [ ] Validate witness format compatibility

2. **Short-term (Week 1-2)**:
   - [ ] Complete research phase
   - [ ] Finalize interface design
   - [ ] Create detailed implementation plan

3. **Medium-term (Week 3-8)**:
   - [ ] Implement core integration
   - [ ] Integrate proof generation
   - [ ] Add configuration support

4. **Long-term (Week 9-12)**:
   - [ ] Complete testing
   - [ ] Performance optimization
   - [ ] Documentation and cleanup

## References

- [ZisKVM Documentation](https://0xpolygonhermez.github.io/zisk/)
- [ZisKVM Quickstart](https://0xpolygonhermez.github.io/zisk/getting_started/quickstart.html)
- [ZisKVM GitHub Repository](https://github.com/0xPolygonHermez/zisk)
- [Plonky3 Documentation](https://github.com/Plonky3/Plonky3)

## Appendix: Code Locations

### Current FEP Proving System
- Witness Generation: `zk/witness/witness.go`
- DataStream: `zk/datastream/server/data_stream_server.go`
- Legacy Executor: `zk/legacy_executor_verifier/`
- RPC Endpoints: `turbo/jsonrpc/zkevm_api.go`
- Stage Execution: `zk/stages/stage_sequence_execute.go`

### Proposed ZisKVM Integration
- ZisKVM Prover: `zk/ziskvm_prover/` (new)
- Configuration: `eth/ethconfig/config_zkevm.go`
- CLI Flags: `cmd/utils/flags.go`
- Integration Points: `zk/stages/stage_sequence_execute.go`, `turbo/jsonrpc/zkevm_api.go`
