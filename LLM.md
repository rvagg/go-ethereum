# LLM Analysis Context - Ethereum API Compatibility

This document contains context for LLM assistants to help analyze compatibility issues between Ethereum JSON-RPC API implementations. It represents the current state of analysis as of April 2025 on compatibility between Lotus's Ethereum API implementation, go-ethereum's native API implementation, and Erigon's implementation.

## Project Overview

This analysis compares three Ethereum-compatible JSON-RPC API implementations:
1. **go-ethereum**: The reference Ethereum client implementation
2. **Lotus**: Filecoin's Ethereum compatibility layer 
3. **Erigon**: An alternative Ethereum client with optimizations

The main documentation is in `LOTUS_ETHEREUM.md`, which has been formatted to be compatible with Notion and contains tables comparing method availability, parameter handling, and potential compatibility issues.

## Key Findings

1. **Block Tag Handling Inconsistencies**: 
   - Lotus has inconsistent block tag support across methods:
     - Most methods like `eth_getBlockByNumber` support tags via `getTipsetByBlockNumber`
     - `eth_getBlockTransactionCountByNumber` uses `EthUint64` and doesn't support tags
     - `eth_getLogs` supports only "latest" and "earliest" (not "pending", "safe", "finalized")
   - Erigon adds additional tags like "latestExecuted"
   - Tag meanings differ - "safe" and "finalized" in Lotus are fixed offsets from latest

2. **Method Availability Gaps**:
   - Lotus doesn't support newer Ethereum features:
     - No EIP-2930 access list support
     - No EIP-4844 blob transaction support
     - No support for uncle-related methods (not applicable to Filecoin)
   - Lotus adds Filecoin-specific methods for CID lookups
   - Erigon adds optimized APIs like bitmap-based log filtering

3. **Parameter Handling Differences**:
   - go-ethereum is strictest (hex only with "0x", no leading zeros)
   - Lotus is most flexible (decimal or hex, accepts leading zeros)
   - Erigon tries decimal first, then hex

4. **Transaction Types**:
   - go-ethereum supports all transaction types
   - Lotus mainly supports EIP-1559 (dynamic fee) transactions
   - Applications using newer transaction types may fail on Lotus

## Recent Discoveries

Recent investigation revealed that:
1. Lotus does implement `eth_getTransactionByBlockHashAndIndex` and `eth_getTransactionByBlockNumberAndIndex` (previously marked as missing)
2. Lotus has very limited block tag support in `eth_getLogs` - only "latest" and "earliest" are supported based on analysis of the `parseBlockRange` function in `node/impl/eth/events.go`
3. The `getTipsetByBlockNumber` helper in `node/impl/eth/utils.go` explicitly does not support the "earliest" tag and returns an error
4. Lotus has a split implementation of block tag handling:
   - The JSON unmarshaling of `EthBlockNumberOrHash` only supports "earliest", "latest", and "pending" tags
   - "safe" and "finalized" tags are handled at the application level in various methods
   - This creates inconsistent behavior as some tags may work in some contexts but not others

## Key Code References

### Lotus Block Tag JSON Unmarshal Implementation

```go
// In Lotus: chain/types/ethtypes/eth_types.go
func (e *EthBlockNumberOrHash) UnmarshalJSON(data []byte) error {
    // Try to unmarshal as EthUint64
    var num EthUint64
    err := num.UnmarshalJSON(data)
    if err == nil {
        e.BlockNumber = &num
        return nil
    }

    // Try to unmarshal as predefined string
    var str string
    err = json.Unmarshal(data, &str)
    if err == nil {
        // Only supports these three tags at JSON level!
        if str == "earliest" || str == "pending" || str == "latest" {
            e.PredefinedBlock = &str
            return nil
        }

        // check if input is a block hash (66 characters long)
        if len(str) == 66 && strings.HasPrefix(str, "0x") {
            hash, err := ParseEthHash(str)
            if err != nil {
                return err
            }
            e.BlockHash = &hash
            return nil
        }
    }

    // Try to unmarshal as a struct
    var bnh struct {
        BlockNumber     *EthUint64 `json:"blockNumber,omitempty"`
        BlockHash       *EthHash   `json:"blockHash,omitempty"`
        RequireCanonical *bool      `json:"requireCanonical,omitempty"`
    }

    err = json.Unmarshal(data, &bnh)
    if err != nil {
        return err
    }

    e.BlockNumber = bnh.BlockNumber
    e.BlockHash = bnh.BlockHash
    e.RequireCanonical = bnh.RequireCanonical

    return nil
}
```

### Lotus Block Tag Handling
```go
// In Lotus: node/impl/eth/utils.go
func getTipsetByBlockNumber(ctx context.Context, cs ChainStore, blkParam string, strict bool) (*types.TipSet, error) {
    if blkParam == "earliest" {
        return nil, xerrors.New("block param \"earliest\" is not supported")
    }

    head := cs.GetHeaviestTipSet()
    switch blkParam {
    case "pending":
        return head, nil
    case "latest":
        parent, err := cs.GetTipSetFromKey(ctx, head.Parents())
        if err != nil {
            return nil, xerrors.New("cannot get parent tipset")
        }
        return parent, nil
    case "safe":
        latestHeight := head.Height() - 1
        safeHeight := latestHeight - ethtypes.SafeEpochDelay
        ts, err := cs.GetTipsetByHeight(ctx, safeHeight, head, true)
        /* ... */
    case "finalized":
        latestHeight := head.Height() - 1
        safeHeight := latestHeight - policy.ChainFinality
        /* ... */
    default:
        var num ethtypes.EthUint64
        err := num.UnmarshalJSON([]byte(`"` + blkParam + `"`))
        /* ... */
    }
}
```

### Lotus eth_getLogs Implementation
```go
// In Lotus: node/impl/eth/events.go
func parseBlockRange(heaviest abi.ChainEpoch, fromBlock, toBlock *string, maxRange abi.ChainEpoch) (minHeight abi.ChainEpoch, maxHeight abi.ChainEpoch, err error) {
    if fromBlock == nil || *fromBlock == "latest" || len(*fromBlock) == 0 {
        minHeight = heaviest
    } else if *fromBlock == "earliest" {
        minHeight = 0
    } else {
        if !strings.HasPrefix(*fromBlock, "0x") {
            return 0, 0, xerrors.New("FromBlock is not a hex")
        }
        epoch, err := ethtypes.EthUint64FromHex(*fromBlock)
        /* ... */
    }

    if toBlock == nil || *toBlock == "latest" || len(*toBlock) == 0 {
        // here latest means the latest at the time
        maxHeight = -1
    } else if *toBlock == "earliest" {
        maxHeight = 0
    } else {
        if !strings.HasPrefix(*toBlock, "0x") {
            return 0, 0, xerrors.New("ToBlock is not a hex")
        }
        /* ... */
    }
    // No support for "pending", "safe", "finalized" in this method
    /* ... */
}
```

### go-ethereum Block Number Handling
```go
// In go-ethereum: rpc/types.go
func (bn *BlockNumber) UnmarshalJSON(data []byte) error {
    input := strings.TrimSpace(string(data))
    if len(input) >= 2 && input[0] == '"' && input[len(input)-1] == '"' {
        input = input[1 : len(input)-1]
    }

    switch input {
    case "earliest":
        *bn = EarliestBlockNumber
        return nil
    case "latest":
        *bn = LatestBlockNumber
        return nil
    case "pending":
        *bn = PendingBlockNumber
        return nil
    case "finalized":
        *bn = FinalizedBlockNumber
        return nil
    case "safe":
        *bn = SafeBlockNumber
        return nil
    }

    // Try to parse as a hex number
    if !strings.HasPrefix(input, "0x") {
        return errors.New("hex number without 0x prefix")
    }
    input = input[2:]
    if len(input) > 0 && input[0] == '0' {
        return errors.New("hex number with leading zero digits")
    }
    /* ... */
}
```

## Repository Structure

Key files and locations for analysis:

### go-ethereum
* `./internal/ethapi/api.go` - Core API implementations
* `./rpc/types.go` - JSON-RPC types like BlockNumber
* `./common/types.go` - Core types like Hash
* `./eth/filters/api.go` - Filter API implementation

### Lotus
* `./lotus/node/impl/full/eth.go` - Ethereum API implementations
* `./lotus/node/impl/eth/*.go` - Implementation files for specific APIs
* `./lotus/chain/types/ethtypes/eth_types.go` - Ethereum compatibility types
* `./lotus/api/eth_aliases.go` - Maps Lotus methods to JSON-RPC names

### Erigon
* `./erigon/turbo/jsonrpc/eth_*.go` - Ethereum API implementations
* `./erigon/turbo/jsonrpc/trace_*.go` - Trace API implementations
* `./erigon/rpc/types.go` - JSON-RPC types
* `./erigon/erigon-lib/common/hash.go` - Core types like Hash

Each implementation has its own approach to the Ethereum JSON-RPC API, with varying levels of compatibility, parameter handling, and extension methods. This document details the similarities and differences across these implementations to help understand potential compatibility challenges.

## Fundamental Architectural Differences

### Blockchain Structure
* **go-ethereum**: Linear blockchain with single blocks
* **Erigon**: Same as go-ethereum, but with optimized storage architecture
* **Lotus**: Filecoin uses tipsets (groups of blocks at same height)

### Address System
* **go-ethereum**: Native 20-byte Ethereum addresses (0x prefix)
* **Erigon**: Same as go-ethereum
* **Lotus**: Maps between multiple Filecoin address types (f0, f1, f2, f3, f4) and Ethereum addresses

### Transaction Model
* **go-ethereum**: Multiple native transaction types (legacy, EIP-1559, EIP-2930, EIP-4844)
* **Erigon**: Same transaction types as go-ethereum, minus some newer types
* **Lotus**: Converts Filecoin messages to Ethereum-style transactions (mostly as EIP-1559)

### Consensus
* **go-ethereum**: Originally PoW, now PoS with beacon chain
* **Erigon**: Same as go-ethereum, plus compatibility with Polygon/Bor protocol
* **Lotus**: Filecoin consensus with different finality rules

### State Access
* **go-ethereum**: Direct access to Ethereum state
* **Erigon**: Optimized state access with specialized indices
* **Lotus**: Translates Ethereum state queries to Filecoin state tree operations

### Storage Architecture
* **go-ethereum**: LevelDB-based storage with a single state trie
* **Erigon**: Highly optimized storage with separate indices for different data types
* **Lotus**: Filecoin-specific storage architecture with different state model

## Core Type Implementation Comparison

| Type | go-ethereum | Erigon | Lotus |
|------|-------------|--------|-------|
| **BlockNumber** | int64 constants:<br>- EarliestBlockNumber = -5<br>- SafeBlockNumber = -4<br>- FinalizedBlockNumber = -3<br>- LatestBlockNumber = -2<br>- PendingBlockNumber = -1 | int64 constants with different values:<br>- LatestExecutedBlockNumber = -5<br>- FinalizedBlockNumber = -4<br>- SafeBlockNumber = -3<br>- PendingBlockNumber = -2<br>- LatestBlockNumber = -1<br>- EarliestBlockNumber = 0 | String representations:<br>- "earliest"<br>- "latest"<br>- "pending"<br>- "safe" (30 epochs behind)<br>- "finalized" (900 epochs behind) |
| **Number Format** | - Hex-only with "0x" prefix<br>- Rejects leading zeros<br>- Strict validation | - Decimal or hex format<br>- Tries decimal first, then hex<br>- More lenient validation | - Decimal or hex format<br>- Optional "0x" prefix for decimal<br>- Accepts leading zeros |
| **Hash** | - 32-byte array<br>- Requires "0x" prefix<br>- Strict validation | - 32-byte array<br>- Requires "0x" prefix<br>- Similar validation to go-ethereum | - 32-byte array<br>- Requires "0x" prefix<br>- Converts to/from Filecoin CIDs |
| **Special Tags** | - "earliest", "latest", "pending"<br>- Post-Merge: "safe", "finalized" | - Same as go-ethereum<br>- Adds: "latestExecuted", "null" | - Same tags as go-ethereum<br>- Different interpretations for "safe" and "finalized" |

## JSON-RPC Method Comparison

### Method Organization Patterns

#### go-ethereum
1. Standard Ethereum methods in eth_ namespace
2. Debug/trace methods in debug_ namespace
3. Extension methods in various namespaces (txpool_, miner_, etc.)

#### Erigon
1. Standard Ethereum methods in eth_ namespace
2. Trace methods in trace_ namespace (OpenEthereum-compatible)
3. Extension methods in erigon_ namespace
4. Otterscan integration in ots_ namespace 

#### Lotus
1. Standard Ethereum methods in eth_ namespace
2. Tracing methods in trace_ namespace (mapped to Filecoin.Eth* internally)
3. Lotus-specific "Limited" variants of standard methods
4. Filecoin-specific methods for CID/transaction hash mapping

## Block Specifier Handling

### Block Number Unmarshaling

| Feature | go-ethereum | Lotus |
|---------|-------------|-------|
| **Numeric Format** | Hex only | Both hex and decimal |
| **0x Prefix** | Required for all numeric inputs | Optional for decimal, required for hex |
| **Leading Zeros** | Rejected (e.g., "0x01" fails) | Accepted |
| **Raw JSON Numbers** | Not accepted | Accepted (e.g., `42` without quotes) |
| **Special Tags** | Handled in BlockNumber | Handled in EthBlockNumberOrHash |
| **Range Limits** | Rejects values > int64 max | uint64 range |
| **Inconsistent Support** | Consistent tag support across all methods | Inconsistent: Some methods use EthUint64 which doesn't support tags |

### Block Tag Support in Lotus Methods

Lotus has mixed support for block tags:

1. **Methods with Full Block Tag Support**:
   - `eth_getBlockByNumber` - Takes a string parameter and handles "latest", "pending", "safe", "finalized" tags
   - `eth_getTransactionByBlockNumberAndIndex` - Takes a string parameter and handles the same tags
   - Most other methods that use the `getTipsetByBlockNumber` helper support these tags

2. **Methods With Limited Block Tag Support**:
   - `eth_getLogs` - Only supports "latest" and "earliest" in its FromBlock/ToBlock parameters
   - Doesn't support "pending", "safe", or "finalized" tags in log filters

3. **Methods Without Block Tag Support**:
   - `eth_getBlockTransactionCountByNumber` - Uses `EthUint64`, only accepts numeric values
   
Many Lotus methods use the `getTipsetByBlockNumber` helper function to handle block tags, which converts string tags to appropriate Filecoin tipsets. Note that "earliest" tag is explicitly not supported and returns an error in methods that use this helper.

### Block Number or Hash Unmarshaling

| Feature | go-ethereum | Lotus |
|---------|-------------|-------|
| **Special Tags** | Supports "earliest", "latest", "pending", "safe", "finalized" | Supports all five tags with different meanings |
| **Internal Representation** | Special tags become negative integers | Special tags stored as string pointers |
| **Block Hash Validation** | 66 chars AND must have "0x" prefix | 66 chars AND must have "0x" prefix |
| **JSON Field Names** | Case-sensitive | Case-sensitive |

### Block Tag Interpretation

| Block Tag | go-ethereum | Lotus |
|-----------|-------------|-------|
| **"earliest"** | Genesis block (block 0) | Genesis block (block 0) |
| **"latest"** | Most recent mined block | Most recent mined block (parent of tipset) |
| **"pending"** | Current state with pending transactions | Current state with pending messages |
| **"safe"** | Safe head from consensus layer | 30 epochs behind "latest" (SafeEpochDelay) |
| **"finalized"** | Finalized head from consensus layer | 900 epochs behind "latest" (ChainFinality) |

## Method Naming and Mapping

Lotus implements Ethereum-compatible methods but uses specific mapping rules:

1. Standard Ethereum methods use the standard namespace (e.g., `eth_blockNumber`, `eth_call`)
2. Tracing methods use the `trace_` namespace instead of go-ethereum's `debug_` namespace:
   * `trace_block` → Maps to `Filecoin.EthTraceBlock` internally
   * `trace_filter` → Maps to `Filecoin.EthTraceFilter` internally 
   * `trace_replayBlockTransactions` → Maps to `Filecoin.EthTraceReplayBlockTransactions` internally
   * `trace_transaction` → Maps to `Filecoin.EthTraceTransaction` internally
3. All methods are internally mapped to `Filecoin.*` namespace as defined in `eth_aliases.go`

## Method-Specific Differences

### `eth_getBlockByNumber` and related methods

* **go-ethereum**: 
  - Requires "0x" prefix for numbers, rejects leading zeros
  - Maps special tags to negative integers internally
  - Returns fully populated block structure

* **Erigon**:
  - Accepts both decimal and hex format for block numbers
  - Supports additional tags: "latestExecuted" and "null"
  - Special handling for pending blocks (nil-out fields)
  - Block caching using LRU for performance

* **Lotus**:
  - Accepts decimal format, accepts leading zeros
  - Different interpretation of "safe" (30 epochs) and "finalized" (900 epochs)
  - Creates placeholder values for many Ethereum fields

### `eth_getBlockByHash` and hash-based methods

* **go-ethereum**:
  - Requires "0x" prefix for hashes
  - Returns error for non-existent blocks
  - Standard Ethereum block structure 

* **Erigon**:
  - Requires "0x" prefix for hashes
  - Special handling for clients using wrong method type
  - Returns nil instead of error for non-existent blocks
  - Cache handling and optimization

* **Lotus**:
  - Requires "0x" prefix for hashes
  - Converts between Ethereum hashes and Filecoin CIDs
  - Simulates Ethereum block from Filecoin tipset

### `eth_call` and state access methods

* **go-ethereum**:
  - Standard BlockNumberOrHash parameter
  - Strict validation of transaction parameters
  - Direct access to EVM for execution

* **Erigon**:
  - Similar to go-ethereum but with more validation
  - Sets default gas value explicitly
  - Special limits on return data size
  - Canonical block enforcement

* **Lotus**:
  - Translates Ethereum address formats to Filecoin
  - Different state access mechanism using Filecoin VM
  - Similar parameter interface but different execution

### `eth_getLogs` and filter methods

* **go-ethereum**:
  - Standard FilterCriteria with topic filtering
  - Supports all block tags ("earliest", "latest", "pending", "safe", "finalized")
  - Converts block tags to negative integers internally
  - Block range and block hash support
  - Detailed validation logic

* **Erigon**:
  - Highly optimized using bitmap operations
  - More efficient topic filtering
  - Special handling for Bor protocol (Polygon)
  - Advanced implementation details for performance

* **Lotus**:
  - Similar interface but translated to Filecoin events
  - Limited block tag support - only "latest" and "earliest"
  - Does NOT support "pending", "safe", or "finalized" in log filters
  - Custom parseBlockRange function with different validation logic
  - Enforces maximum range limitation 
  - Topic conversion between Ethereum and Filecoin formats

### Block Transaction Methods

* **go-ethereum**:
  - Full implementation of block and transaction access methods:
    * eth_getTransactionByBlockHashAndIndex
    * eth_getTransactionByBlockNumberAndIndex
    * eth_getRawTransactionByBlockHashAndIndex
    * eth_getRawTransactionByBlockNumberAndIndex
  - Directly maps to native Ethereum block and transaction structures

* **Erigon**:
  - Full implementation matching go-ethereum's capabilities
  - Possible optimizations for performance in high load scenarios

* **Lotus**:
  - Implements core transaction access methods:
    * eth_getTransactionByBlockHashAndIndex
    * eth_getTransactionByBlockNumberAndIndex
  - Does not support raw transaction variants
  - Translated from Filecoin tipsets and messages to Ethereum format
  - Maps between Ethereum block hashes and Filecoin CIDs internally

### Tracing methods

* **go-ethereum**:
  - Uses `debug_trace*` namespace
  - Integrated with EVM for detailed tracing
  - Formats results according to Ethereum conventions

* **Erigon**:
  - Uses `trace_*` namespace (OpenEthereum-compatible)
  - Comprehensive trace API with additional methods
  - Specialized optimization and execution

* **Lotus**:
  - Uses `trace_*` namespace (mapped to Filecoin.Eth* internally)
  - Different tracing implementation based on Filecoin VM
  - Simulates Ethereum trace format

### Transaction processing

* **go-ethereum**: 
  - Full support for all transaction types (legacy, EIP-1559, EIP-2930, EIP-4844)
  - Complete transaction signing and validation
  - Native EVM execution
  - Full implementation of gas pricing methods (eth_feeHistory, eth_maxPriorityFeePerGas)

* **Erigon**:
  - Support for transaction types including legacy, EIP-1559, EIP-2930
  - Full support for eth_feeHistory and eth_maxPriorityFeePerGas
  - May have different implementation details but compatible interfaces
  - Optimized transaction pool and processing

* **Lotus**: 
  - Limited support, primarily EIP-1559 transactions
  - Converts between Filecoin messages and Ethereum transactions
  - No support for access lists or blob transactions
  - Implements eth_feeHistory and eth_maxPriorityFeePerGas by mapping Filecoin gas concepts to Ethereum
## Response Structure Differences

### Block Structure

* **go-ethereum**: 
  - Complete Ethereum blocks with all fields
  - Proper consensus fields (PoW or PoS)
  - All transaction types properly represented
  - Full uncle information
  - Post-Merge and post-Cancun fields when applicable

* **Erigon**:
  - Complete Ethereum blocks with all fields
  - Same as go-ethereum with minor formatting differences
  - Special handling for Polygon/Bor protocol blocks when applicable

* **Lotus**: 
  - Simulated Ethereum blocks from Filecoin tipsets
  - Placeholder values for many fields:
    * nonce=0, difficulty=0, mixHash empty
    * sha3Uncles is constant empty hash
    * No uncle-related fields
    * No post-Merge fields (withdrawals, parentBeaconBlockRoot)
    * No post-Cancun blob gas fields

### Transaction Structure

* **go-ethereum**: 
  - Appropriate type representation for each transaction type
  - Full transaction metadata
  - Proper signature components

* **Erigon**:
  - Similar to go-ethereum
  - Optimized storage and retrieval
  - May lack newest transaction type support

* **Lotus**: 
  - All transactions represented as EIP-1559 regardless of original type
  - Converted from Filecoin messages
  - Different block inclusion semantics due to tipsets vs blocks

### Receipt Structure

* **go-ethereum**:
  - Standard Ethereum receipt format
  - Proper status codes and event logs
  - Transaction type-specific fields

* **Erigon**:
  - Same format as go-ethereum
  - Optimized storage and retrieval
  - Possibly additional information in some cases

* **Lotus**:
  - Simulated from Filecoin execution results
  - Subtle differences in gas calculation and log format
  - Gas used and effective gas price calculated differently
  - Logs based on Filecoin events vs Ethereum logs

## Key Compatibility Issues to Consider

### Across All Implementations

1. **Block Tag Interpretation**: 
   - Different levels of tag support:
     * go-ethereum: Consistent support for "earliest", "latest", "pending", "safe", "finalized" at JSON level
     * Erigon: Adds "latestExecuted" and "null" tags
     * Lotus: **Split implementation** - JSON unmarshal only supports "earliest", "latest", "pending", while "safe"/"finalized" are handled at application level
   - Different meanings for "safe" and "finalized":
     * go-ethereum: Based on Ethereum consensus layer
     * Erigon: Based on Ethereum consensus layer
     * Lotus: "safe" = 30 epochs behind latest, "finalized" = 900 epochs behind latest
   - **Critical note**: Some methods like `getTipsetByBlockNumber` explicitly reject "earliest" tag despite it being supported at JSON level
   - Different methods have inconsistent tag support (e.g., eth_getBlockTransactionCountByNumber uses EthUint64 and has no tag support)

2. **Block Number Format Requirements**: 
   - go-ethereum: Strict hex with "0x" prefix, rejects leading zeros
   - Erigon: Flexible, accepts decimal (tries first) and hex
   - Lotus: Accepts both decimal and hex, optional "0x" prefix for decimal

3. **Error Handling Differences**: 
   - go-ethereum: Returns errors for not-found resources
   - Erigon: Often returns nil for not-found resources
   - Lotus: Custom error handling related to Filecoin architecture

4. **Method Namespace Differences**: 
   - go-ethereum: Uses debug_trace* for tracing
   - Erigon: Uses trace_* for tracing (OpenEthereum compatibility)
   - Lotus: Uses trace_* mapped to Filecoin.Eth* internally

### go-ethereum vs. Erigon

1. **BlockNumber Constants**: 
   - Different values for internal constants:
     * go-ethereum: EarliestBlockNumber = -5, SafeBlockNumber = -4, etc.
     * Erigon: LatestExecutedBlockNumber = -5, FinalizedBlockNumber = -4, etc.
   - Erigon defines EarliestBlockNumber as 0 (not negative)

2. **Additional Tags and Functionality**: 
   - Erigon supports extra tags like "latestExecuted" and "null"
   - Erigon has specialized optimizations for Polygon/Bor protocol
   - Erigon implements bitmap-based log filtering for performance

3. **Error Handling Pattern**: 
   - Erigon often returns nil for not-found blocks
   - go-ethereum returns errors for not-found resources
   - Erigon has special handling for client peculiarities (e.g., eth_getBlockByHash with number)

4. **Extension Methods**: 
   - Erigon: Uses erigon_* namespace for extensions (getBlockByTimestamp, getBalanceChangesInBlock)
   - go-ethereum: Various extension namespaces for different functionality

5. **Tracing Approach**: 
   - Erigon implements the OpenEthereum trace_* API
   - go-ethereum uses debug_trace* namespace
   - Erigon has additional trace methods like trace_callMany

### go-ethereum/Erigon vs. Lotus

1. **Transaction Type Support**: 
   - Lotus primarily supports EIP-1559 transactions
   - No support for EIP-2930 (access lists) or EIP-4844 (blob transactions)
   - Converts Filecoin messages to Ethereum transaction format

2. **Gas Model and Fee Differences**: 
   - Lotus maps Filecoin gas concepts to Ethereum equivalents
   - Different calculation methods affect gas estimation and fees
   - eth_feeHistory and eth_maxPriorityFeePerGas implemented but with Filecoin semantics

3. **Response Structure Limitations**: 
   - Missing/placeholder fields in Lotus responses:
     * nonce=0, difficulty=0, mixHash empty
     * sha3Uncles is constant empty hash
     * No uncle-related fields
     * No post-Merge fields (withdrawals, parentBeaconBlockRoot)
     * No post-Cancun blob gas fields

4. **Architectural Translation Layer**: 
   - Filecoin tipsets mapped to Ethereum blocks
   - Filecoin messages mapped to Ethereum transactions
   - Filecoin events mapped to Ethereum logs
   - Filecoin state mapped to Ethereum state

5. **Address Translation Complexities**: 
   - Maps between multiple Filecoin address types (f0, f1, f2, f3, f4) and Ethereum addresses
   - Special handling for delegated addresses and actor IDs
   - Potential edge cases with deleted actors

6. **Range and Performance Limitations**: 
   - Lotus enforces stricter range limitations for queries like eth_getLogs
   - Different performance characteristics due to Filecoin's different architecture
   - "Limited" variants of methods with explicit depth/complexity restrictions

## Example Incompatibility Scenarios

1. **Block Tag Semantic Differences**:
   - Tools relying on the canonical meanings of "safe" and "finalized" post-Merge will behave differently with Lotus
   - Applications using Erigon's "latestExecuted" tag will fail on go-ethereum or Lotus

2. **Transaction Type Compatibility**:
   - Applications using EIP-2930 access list transactions will fail on Lotus
   - Applications using EIP-4844 blob transactions will fail on Lotus and potentially on Erigon
   - Applications relying on exact transaction type encoding details may experience issues

3. **Block Structure Expectations**:
   - Tools expecting uncle-related fields will encounter empty/null values with Lotus
   - Applications relying on post-Merge or post-Cancun fields will find them missing in Lotus
   - Tools making assumptions about nonce, difficulty, or other consensus fields will see unexpected values in Lotus

4. **Parameter Format and Block Tag Handling**:
   - Tools that submit block numbers without "0x" prefix will fail with go-ethereum but work with Erigon and Lotus
   - Applications submitting block numbers with leading zeros will fail with go-ethereum but work with the others
   - Tools relying on decimal block numbers will have issues with go-ethereum
   - Applications using "safe" or "finalized" tags at the JSON level will fail with Lotus's EthBlockNumberOrHash.UnmarshalJSON
   - Applications using "earliest" tag with getTipsetByBlockNumber will receive an error despite the tag being accepted at JSON level

5. **Gas and Fee Calculation**:
   - Applications relying on exact gas calculation semantics will see different results across implementations
   - Smart contracts with tight gas requirements may behave differently
   - Fee history and estimation tools may show differing results

6. **Error Handling Expectations**:
   - Web3 tools that expect specific error responses for non-existent blocks will behave differently with Erigon
   - Applications that don't handle different error formats gracefully may break

7. **Method Availability Differences**:
   - Applications relying on raw transaction methods (eth_getRawTransactionByHash, eth_getRawTransactionByBlockHashAndIndex) will fail on Lotus
   - Applications requiring uncle-related methods will fail on Lotus
   - Tools expecting full spec-compliant header methods may experience issues
   
8. **Address and State Translation**:
   - Ethereum tools interacting with Filecoin-native addresses may encounter translation edge cases
   - Applications assuming Ethereum's state model may have subtle issues with Lotus

9. **Range and Performance**:
   - High-volume log queries may hit Lotus's range limitations
   - Applications optimized for a specific implementation's performance characteristics might perform poorly on others

This document represents our detailed analysis of compatibility between go-ethereum, Erigon, and Lotus Ethereum API implementations. It serves as a comprehensive reference for understanding the subtle differences that may affect applications moving between these systems or attempting to build cross-blockchain compatible tools.