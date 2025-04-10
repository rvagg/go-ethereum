# Comprehensive Ethereum API Compatibility Analysis

This document captures the complete current state of analysis on the compatibility between Lotus's Ethereum API implementation, go-ethereum's native API implementation, and Erigon's implementation. Use this as context for future conversations to continue our detailed comparison.

## Introduction

This analysis compares three Ethereum-compatible JSON-RPC API implementations:

1. **go-ethereum**: The reference Ethereum client implementation
2. **Lotus**: Filecoin's Ethereum compatibility layer that allows Ethereum tools to interact with the Filecoin blockchain
3. **Erigon**: An alternative Ethereum client implementation with performance optimizations and extensions

Each implementation has its own approach to the Ethereum JSON-RPC API, with varying levels of compatibility, parameter handling, and extension methods. This document details the similarities and differences across these implementations to help understand potential compatibility challenges.

## Repository Context

* `/home/rvagg/go/src/github.com/ethereum/go-ethereum/` - The go-ethereum repository
* `./lotus/` - A symlink to the Filecoin Lotus repository
* `./erigon/` - A clone of the Erigon repository

### Key Files

#### go-ethereum
* `./internal/ethapi/api.go` - Core Ethereum API implementations
* `./rpc/types.go` - Types like BlockNumber and BlockNumberOrHash
* `./common/types.go` - Core types like Hash
* `./common/hexutil/hexutil.go` - Hex value parsing
* `./eth/filters/api.go` - Filter API implementation

#### Lotus
* `./lotus/node/impl/full/eth.go` - Ethereum API implementations
* `./lotus/chain/types/ethtypes/eth_types.go` - Ethereum compatibility types
* `./lotus/api/eth_aliases.go` - Maps Lotus's Ethereum-compatible methods to JSONRPC method names

#### Erigon
* `./erigon/turbo/jsonrpc/eth_*.go` - Ethereum API implementation files
* `./erigon/turbo/jsonrpc/trace_*.go` - Trace API implementation files
* `./erigon/rpc/types.go` - JSON-RPC types
* `./erigon/erigon-lib/common/hash.go` - Core types like Hash
* `./erigon/turbo/jsonrpc/daemon.go` - Namespace registration

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
   - All support "latest", "pending", "safe", "finalized"
   - Different meanings for "safe" and "finalized":
     * go-ethereum: Based on Ethereum consensus layer
     * Erigon: Based on Ethereum consensus layer
     * Lotus: "safe" = 30 epochs behind latest, "finalized" = 900 epochs behind latest
   - Erigon adds "latestExecuted" and "null" tags
   - go-ethereum and Erigon support "earliest", but Lotus explicitly does not
   - **Note**: Most Lotus methods support block tags, but there are exceptions like eth_getBlockTransactionCountByNumber

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

4. **Parameter Format Requirements**:
   - Tools that submit block numbers without "0x" prefix will fail with go-ethereum but work with Erigon and Lotus
   - Applications submitting block numbers with leading zeros will fail with go-ethereum but work with the others
   - Tools relying on decimal block numbers will have issues with go-ethereum

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