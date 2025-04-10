# Ethereum API Compatibility Analysis

## Context

This document compares the Ethereum JSON-RPC API implementations across three codebases:

* **go-ethereum**: The official Ethereum client implementation
  * `./internal/ethapi/api.go` - Core API implementations
  * `./rpc/types.go` - JSON-RPC types like BlockNumber
  * `./common/types.go` - Core types like Hash

* **Lotus**: Filecoin's Ethereum compatibility layer
  * `./lotus/node/impl/full/eth.go` - Ethereum API implementations
  * `./lotus/chain/types/ethtypes/eth_types.go` - Ethereum-compatible types
  * `./lotus/api/eth_aliases.go` - Maps methods to JSON-RPC names

* **Erigon**: An alternative Ethereum client implementation
  * `./erigon/turbo/jsonrpc/eth_*.go` - Ethereum API implementations
  * `./erigon/rpc/types.go` - JSON-RPC types
  * `./erigon/erigon-lib/common/hash.go` - Core types like Hash

## Core Type Implementation Comparison

| Type | go-ethereum | Erigon | Lotus | Key Differences |
|------|-------------|--------|-------|-----------------|
| **BlockNumber** | int64 with special constants:<br>- `EarliestBlockNumber` = -5<br>- `SafeBlockNumber` = -4<br>- `FinalizedBlockNumber` = -3<br>- `LatestBlockNumber` = -2<br>- `PendingBlockNumber` = -1 | int64 with different constants:<br>- `LatestExecutedBlockNumber` = -5<br>- `FinalizedBlockNumber` = -4<br>- `SafeBlockNumber` = -3<br>- `PendingBlockNumber` = -2<br>- `LatestBlockNumber` = -1<br>- `EarliestBlockNumber` = 0 | Uses string representation in `EthBlockNumberOrHash`:<br>- `"earliest"`<br>- `"latest"`<br>- `"pending"`<br>- `"safe"` (30 epochs behind)<br>- `"finalized"` (900 epochs behind)<br>**Important**: Some methods use `EthUint64` which doesn't support tags | - Erigon has different constant values<br>- Erigon adds `LatestExecutedBlockNumber`<br>- Lotus uses string representation<br>- Different interpretations of "safe"/"finalized"<br>- Lotus has inconsistent block tag support |
| **Number Format** | Hex-only, requires "0x" prefix, rejects leading zeros | Accepts both decimal and hex, tries decimal first | Accepts both decimal and hex, optional "0x" prefix for decimal | - go-ethereum is strictest<br>- Erigon and Lotus are more flexible |
| **Hash** | Fixed 32-byte array, requires "0x" prefix | Fixed 32-byte array, requires "0x" prefix | Fixed 32-byte array, requires "0x" prefix | - Very similar implementations<br>- All require "0x" prefix |
| **Special Tags** | "earliest", "latest", "pending", "safe", "finalized" | All go-ethereum tags plus "latestExecuted", "null" | "earliest", "latest", "pending", "safe", "finalized" | - Erigon adds additional tags<br>- Different interpretations of tags |

## JSON-RPC Method Comparison

The tables below compare the Ethereum JSON-RPC methods implemented across the three codebases, organized by category.

### Core Ethereum API Methods

| Method Name | Lotus | go-ethereum | Erigon | Notes |
|-------------|:-----:|:-----------:|:------:|-------|
| eth_accounts | ✅ | ✅ | ✅ | |
| eth_blockNumber | ✅ | ✅ | ✅ | |
| eth_call | ✅ | ✅ | ✅ | |
| eth_chainId | ✅ | ✅ | ✅ | |
| eth_estimateGas | ✅ | ✅ | ✅ | |
| eth_feeHistory | ✅ | ✅ | ✅ | Different implementations with same interface |
| eth_gasPrice | ✅ | ✅ | ✅ | |
| eth_getBalance | ✅ | ✅ | ✅ | |
| eth_getBlockByHash | ✅ | ✅ | ✅ | |
| eth_getBlockByNumber | ✅ | ✅ | ✅ | |
| eth_getBlockReceipts | ✅ | ✅ | ✅ | |
| eth_getBlockTransactionCountByHash | ✅ | ✅ | ✅ | |
| eth_getBlockTransactionCountByNumber | ✅ | ✅ | ✅ | |
| eth_getCode | ✅ | ✅ | ✅ | |
| eth_getStorageAt | ✅ | ✅ | ✅ | |
| eth_getTransactionByHash | ✅ | ✅ | ✅ | |
| eth_getTransactionCount | ✅ | ✅ | ✅ | |
| eth_getTransactionReceipt | ✅ | ✅ | ✅ | |
| eth_maxPriorityFeePerGas | ✅ | ✅ | ✅ | Different implementations with same interface |
| eth_protocolVersion | ✅ | ✅ | ✅ | |
| eth_sendRawTransaction | ✅ | ✅ | ✅ | |
| eth_syncing | ✅ | ✅ | ✅ | |

### Filter and Subscription Methods

| Method Name | Lotus | go-ethereum | Erigon | Notes |
|-------------|:-----:|:-----------:|:------:|-------|
| eth_getLogs | ✅ | ✅ | ✅ | Different implementation in Erigon using bitmaps |
| eth_getFilterChanges | ✅ | ✅ | ✅ | |
| eth_getFilterLogs | ✅ | ✅ | ✅ | |
| eth_newBlockFilter | ✅ | ✅ | ✅ | |
| eth_newFilter | ✅ | ✅ | ✅ | |
| eth_newPendingTransactionFilter | ✅ | ✅ | ✅ | |
| eth_uninstallFilter | ✅ | ✅ | ✅ | |
| eth_subscribe | ✅ | ✅ | ✅ | |
| eth_unsubscribe | ✅ | ✅ | ✅ | |

### Advanced Transaction Methods

| Method Name | Lotus | go-ethereum | Erigon | Notes |
|-------------|:-----:|:-----------:|:------:|-------|
| eth_createAccessList | ❌ | ✅ | ✅ | EIP-2930 related method |
| eth_fillTransaction | ❌ | ✅ | ❌ | go-ethereum specific |
| eth_pendingTransactions | ❌ | ✅ | ❌ | |
| eth_resend | ❌ | ✅ | ❌ | |
| eth_sendTransaction | ❌ | ✅ | ✅ | Requires unlocked account |
| eth_sign | ❌ | ✅ | ✅ | Requires unlocked account |
| eth_signTransaction | ❌ | ✅ | ✅ | Requires unlocked account |
| eth_simulateV1 | ❌ | ✅ | ❌ | |
| eth_submitTransaction | ❌ | ✅ | ❌ | |

### Block Detail Methods

| Method Name | Lotus | go-ethereum | Erigon | Notes |
|-------------|:-----:|:-----------:|:------:|-------|
| eth_getHeaderByHash | ❌ | ✅ | ❌ | Available in Erigon under erigon_getHeaderByHash |
| eth_getHeaderByNumber | ❌ | ✅ | ❌ | Available in Erigon under erigon_getHeaderByNumber |
| eth_getTransactionByBlockHashAndIndex | ✅ | ✅ | ✅ | |
| eth_getTransactionByBlockNumberAndIndex | ✅ | ✅ | ✅ | |
| eth_getRawTransactionByBlockHashAndIndex | ❌ | ✅ | ✅ | |
| eth_getRawTransactionByBlockNumberAndIndex | ❌ | ✅ | ✅ | |
| eth_getRawTransactionByHash | ❌ | ✅ | ✅ | |
| eth_getProof | ❌ | ✅ | ✅ | |

### Uncle-Related Methods

| Method Name | Lotus | go-ethereum | Erigon | Notes |
|-------------|:-----:|:-----------:|:------:|-------|
| eth_getUncleByBlockHashAndIndex | ❌ | ✅ | ✅ | Not applicable to Filecoin |
| eth_getUncleByBlockNumberAndIndex | ❌ | ✅ | ✅ | Not applicable to Filecoin |
| eth_getUncleCountByBlockHash | ❌ | ✅ | ✅ | Not applicable to Filecoin |
| eth_getUncleCountByBlockNumber | ❌ | ✅ | ✅ | Not applicable to Filecoin |

### Blob Transaction Support

| Method Name | Lotus | go-ethereum | Erigon | Notes |
|-------------|:-----:|:-----------:|:------:|-------|
| eth_blobBaseFee | ❌ | ✅ | ❌ | EIP-4844 related method |

### Tracing Methods

| Method Name | Lotus | go-ethereum | Erigon | Notes |
|-------------|:-----:|:-----------:|:------:|-------|
| trace_block | ✅ | ❌ | ✅ | Similar to debug_traceBlockByNumber in go-ethereum |
| trace_filter | ✅ | ❌ | ✅ | |
| trace_replayBlockTransactions | ✅ | ❌ | ✅ | Listed as trace_replayTransaction in Erigon |
| trace_transaction | ✅ | ❌ | ✅ | |
| trace_call | ❌ | ❌ | ✅ | Erigon-specific |
| trace_callMany | ❌ | ❌ | ✅ | Erigon-specific |
| trace_rawTransaction | ❌ | ❌ | ✅ | Erigon-specific |
| trace_get | ❌ | ❌ | ✅ | Erigon-specific |

### Lotus-Specific Methods

| Method Name | Lotus | go-ethereum | Erigon | Notes |
|-------------|:-----:|:-----------:|:------:|-------|
| eth_getBlockReceiptsLimited | ✅ | ❌ | ❌ | Lotus-specific variant with limitations |
| eth_getMessageCidByTransactionHash | ✅ | ❌ | ❌ | Filecoin-specific method for CID lookup |
| eth_getTransactionByHashLimited | ✅ | ❌ | ❌ | Lotus-specific variant with limitations |
| eth_getTransactionHashByCid | ✅ | ❌ | ❌ | Filecoin-specific method for transaction hash lookup by CID |
| eth_getTransactionReceiptLimited | ✅ | ❌ | ❌ | Lotus-specific variant with limitations |

### Erigon-Specific Methods

| Method Name | Lotus | go-ethereum | Erigon | Notes |
|-------------|:-----:|:-----------:|:------:|-------|
| erigon_forks | ❌ | ❌ | ✅ | Erigon extension |
| erigon_getBlockByTimestamp | ❌ | ❌ | ✅ | Erigon extension |
| erigon_getBalanceChangesInBlock | ❌ | ❌ | ✅ | Erigon extension |
| erigon_getLogsByHash | ❌ | ❌ | ✅ | Erigon extension |

## Method Parameter Handling Comparison

The following table compares how key methods handle their parameters across all three implementations:

| Method | go-ethereum | Erigon | Lotus | Key Differences |
|--------|-------------|--------|-------|-----------------|
| **eth_getBlockByNumber** | Block number must be hex with "0x" prefix<br>Rejects leading zeros<br>Special tags map to negative integers | Accepts both decimal and hex format<br>Special tags plus "latestExecuted", "null"<br>Better error handling for clients | Accepts hex with "0x" prefix or decimal<br>Leading zeros accepted<br>Different interpretations of "safe"/"finalized" | Erigon and Lotus are more flexible<br>Different special tag semantics<br>Erigon special client handling |
| **eth_getBlockByHash** | Hash requires "0x" prefix<br>Returns error for non-existent blocks | Hash requires "0x" prefix<br>Returns nil for non-existent blocks<br>Special handling for clients using wrong method | Hash requires "0x" prefix<br>Converts between Ethereum hashes and Filecoin CIDs | Erigon is most forgiving<br>Lotus has extra conversion layer<br>Different error handling patterns |
| **eth_call** | Uses BlockNumberOrHash for block parameter<br>Strict validation of input parameters | Similar, but adds more validation<br>Sets default gas value explicitly<br>Special limits on return data size | Similar, but also translates Ethereum address formats to Filecoin<br>Different state access mechanism | Different default handling<br>Lotus address translation<br>Differing gas models |
| **eth_getLogs** | Uses FilterCriteria with FromBlock/ToBlock<br>BlockHash mutually exclusive with from/to<br>Block numbers must use "0x" prefix<br>Supports all block tags through rpc.BlockNumber | Similar, but highly optimized using bitmaps<br>Has more efficient topic filtering<br>Special handling for Bor protocol | Similar interface but translated to Filecoin events<br>Enforces maximum range limitation<br>Only supports "latest" and "earliest" tags<br>Block numbers must be hex with "0x" prefix<br>Topics converted from Ethereum to Filecoin format | Erigon has bitmap optimization<br>Lotus has range limitation<br>Lotus has limited block tag support<br>Different topic handling<br>Lotus lacks support for "pending", "safe", "finalized" |

### Notes:
1. Lotus implements tracing functionality under the `trace_*` namespace, while go-ethereum implements similar functionality under the `debug_trace*` namespace. Internally, Lotus maps these trace_* methods to Filecoin.Eth* functions.
2. Lotus has several "Limited" variants of methods that are specific to Filecoin's implementation.
3. Filecoin-specific methods like those involving CIDs are not present in go-ethereum.
4. Erigon includes optimizations and extensions like block-by-timestamp lookups and bitmap-based log filtering.
5. Both Erigon and Lotus are more flexible than go-ethereum when handling numeric inputs, accepting decimal numbers.
6. Lotus does not implement uncle-related methods since Filecoin's consensus does not have uncles.
7. Some advanced or newer Ethereum features (like blob transactions support, access lists) are missing from Lotus.
8. **Important**: Some Lotus methods use `EthUint64` instead of a type that supports block tags. This affects:
   - `eth_getBlockTransactionCountByNumber` - Only accepts numeric block numbers, no "latest", etc.
   This method will fail if clients try to use string tags like "latest" or "pending".
   
   However, many key methods like `eth_getBlockByNumber` and `eth_getTransactionByBlockNumberAndIndex` take a string parameter and properly handle block tags through the `getTipsetByBlockNumber` helper function.

## Parameter Handling Differences

The table below details differences in how Lotus and go-ethereum handle incoming JSON-RPC request parameters and types:

| Parameter Type | go-ethereum | Lotus | Potential Issues |
|----------------|-------------|-------|------------------|
| **Block Number Tags** | Supports "earliest", "latest", "pending", "safe", "finalized" in all methods using rpc.BlockNumber type | Inconsistent tag support:<br>- Most methods support tags via string parameters<br>- Some methods use EthUint64 which only accepts numbers<br>- Different interpretations for "safe" and "finalized" | "safe" in Lotus means 30 epochs behind latest, "finalized" is 900 epochs behind latest (Filecoin's finality)<br>Methods using EthUint64 (like eth_getBlockTransactionCountByNumber) will fail with string tags |
| **Hex Values** | Processes values with "0x" prefix, strict validation | Similar processing with "0x" prefix | Lotus might have more lenient validation in some cases |
| **Block Hash Format** | 32-byte hash with "0x" prefix, strict validation | 32-byte hash with "0x" prefix | Validation logic may differ slightly |
| **Transaction Arguments** | Full support for all transaction types (legacy, EIP-1559, EIP-2930, EIP-4844) | Limited support, primarily for EIP-1559 | Applications using newer transaction types may fail |
| **Access Lists** | Full support in transactions | Not supported | EIP-2930 access list transactions will fail |
| **Blob Transactions** | Full support for EIP-4844 | Not supported | Blob transactions will fail |
| **Address Format** | Native Ethereum addresses | Translates between Filecoin and Ethereum address formats | Subtle issues with certain address types, especially with deleted actors |
| **Gas Parameters** | Native Ethereum gas model | Maps Filecoin gas to Ethereum model | May calculate gas differently, affecting estimations |
| **Log Filtering** | Optimized for Ethereum logs | Simulated from Filecoin events | Complex topic filtering may behave differently |

## Block Specifier Handling by Method

The table below details exactly how block specifiers are handled in each relevant API method, comparing Lotus and go-ethereum implementations:

| Method | Parameter in go-ethereum | Parameter in Lotus | Differences and Compatibility Issues |
|--------|--------------------------|-------------------|-------------------------------------|
| **eth_getBlockByNumber** | `BlockNumber` type with:<br>- Special tags: "earliest", "latest", "pending", "safe", "finalized"<br>- Numeric values **require** "0x" prefix (hex only)<br>- Rejects leading zeros (e.g., "0x01") | `string` parameter with:<br>- Special tags: "latest", "pending", "safe", "finalized" (NOT "earliest")<br>- Uses getTipsetByBlockNumber helper<br>- Numeric values accept both decimal and hex<br>- Accepts leading zeros | - "earliest" tag explicitly not supported in Lotus<br>- Different interpretation of "safe" and "finalized"<br>- Lotus accepts decimal formats (go-ethereum rejects)<br>- go-ethereum rejects leading zeros (Lotus accepts)<br>- Lotus converts between Filecoin tipsets and Ethereum blocks |
| **eth_getBlockTransactionCountByNumber** | `BlockNumber` type with:<br>- Special tags support<br>- Same validation as above | `EthUint64` with:<br>- **NO special tag support**<br>- Only numeric values accepted | - Lotus does not support block tags for this method<br>- Will fail if "latest", "pending", etc. are used<br>- Common cause of compatibility issues |
| **eth_getTransactionByBlockNumberAndIndex** | `BlockNumber` type with:<br>- Special tags support<br>- Same validation as above | `string` parameter with:<br>- Full block tag support<br>- Uses getTipsetByBlockNumber helper<br>- Supports "latest", "pending", "safe", "finalized" | - Lotus's implementation does properly handle tags<br>- "earliest" tag is not supported and returns an error<br>- Different interpretation of "safe" and "finalized" |
| **eth_getBlockByHash** | `Hash` type (32 bytes):<br>- Must be 66 chars (including 0x)<br>- Explicitly requires "0x" prefix | `EthHash` type (32 bytes):<br>- Must be 66 chars (including 0x)<br>- Explicitly requires "0x" prefix | - Both implementations require "0x" prefix for hashes<br>- Lotus performs extra conversion between Ethereum hashes and Filecoin CIDs<br>- Similar validation, both reject hashes without "0x" prefix |
| **eth_call** | `BlockNumberOrHash` with:<br>- Block number (hex only with "0x" prefix)<br>- Block hash (requires "0x" prefix)<br>- Special tags<br>- Optional "requireCanonical" parameter | `EthBlockNumberOrHash` with:<br>- Block number (hex or decimal)<br>- Block hash (requires "0x" prefix)<br>- Special tags<br>- Similar "requireCanonical" parameter | - More flexible numeric formats in Lotus<br>- Both require "0x" prefix for hashes<br>- "safe" and "finalized" have different meanings<br>- Internal representation differences |
| **eth_estimateGas** | Optional `BlockNumberOrHash`<br>- Default to "latest"<br>- Same parsing as eth_call | Optional `EthBlockNumberOrHash`<br>- Similar default<br>- Same parsing as eth_call | - Same format differences as eth_call<br>- Different gas model affects estimation results |
| **eth_getBalance** | `BlockNumberOrHash`<br>- Same parsing as eth_call | `EthBlockNumberOrHash`<br>- Same parsing as eth_call | - Same format differences as eth_call<br>- Address translation between Filecoin and Ethereum formats |
| **eth_getStorageAt** | `BlockNumberOrHash`<br>- Same parsing as eth_call | `EthBlockNumberOrHash`<br>- Same parsing as eth_call | - Same format differences as eth_call<br>- Different storage model between Filecoin and Ethereum |
| **eth_getCode** | `BlockNumberOrHash`<br>- Same parsing as eth_call | `EthBlockNumberOrHash`<br>- Same parsing as eth_call | - Same format differences as eth_call<br>- Different code retrieval mechanism |
| **eth_getLogs** | `FilterCriteria` with:<br>- FromBlock/ToBlock accept special tags ("latest", "pending", "earliest", "safe", "finalized")<br>- BlockHash requires "0x" prefix<br>- Block numbers must use "0x" prefix<br>- BlockHash mutually exclusive with FromBlock/ToBlock | `EthFilterSpec` with:<br>- FromBlock/ToBlock accept special tags ("latest", "pending", "earliest", "safe", "finalized")<br>- Technically requires "0x" prefix for numeric block numbers<br>- BlockHash requires "0x" prefix<br>- BlockHash mutually exclusive with FromBlock/ToBlock<br>- Enforces max range limit between FromBlock and ToBlock | - Both implementations support all five block tags<br>- Both require "0x" prefix for hashes<br>- "safe" in Lotus = 30 epochs behind latest<br>- "finalized" in Lotus = 900 epochs behind latest<br>- Lotus adds maximum range limitation<br>- Internal representation differences |
| **eth_getTransactionCount** | `BlockNumberOrHash`<br>- Same parsing as eth_call | `EthBlockNumberOrHash`<br>- Same parsing as eth_call | - Same format differences as eth_call<br>- Different nonce tracking between Filecoin and Ethereum |
| **eth_getBlockTransactionCountByNumber** | `BlockNumber`<br>- Requires "0x" prefix for numbers<br>- Rejects leading zeros | `EthUint64`<br>- Accepts decimal or hex<br>- Accepts leading zeros | - Lotus accepts more block number formats<br>- Different mechanism for counting transactions (messages in Filecoin) |
| **eth_getBlockTransactionCountByHash** | `Hash`<br>- Explicitly requires "0x" prefix | `EthHash`<br>- Explicitly requires "0x" prefix | - Both implementations require "0x" prefix<br>- Similar hash validation requirements |
| **eth_getTransactionByHash** | No block parameter | No block parameter | - N/A for block specifiers, but transaction format differences exist |
| **eth_feeHistory** | Takes:<br>- Block count<br>- Newest block (with "0x" required)<br>- Reward percentiles | Similar parameters with:<br>- More flexible block number format | - Lotus accepts more block number formats<br>- Fee calculation differences due to different gas models |
| **trace_block** | N/A (debug namespace in go-ethereum) | `EthBlockNumberOrHash` with same parsing as eth_call | - Different namespaces (trace_ vs debug_)<br>- Maps to Filecoin.EthTraceBlock internally<br>- Different tracing implementations |

## Detailed Block Parameter Unmarshaling Comparison

### Block Number Unmarshaling

| Feature | go-ethereum (BlockNumber) | Lotus (EthUint64) | Compatibility Issues |
|---------|---------------------------|-------------------|----------------------|
| **Numeric Format** | Hex only | Both hex and decimal | Decimal numbers accepted by Lotus but rejected by go-ethereum |
| **0x Prefix** | Required for all numeric inputs | Optional for hex, not used for decimal | `"42"` works in Lotus (decimal) but fails in go-ethereum (missing 0x) |
| **Leading Zeros** | Rejected (e.g., "0x01" fails) | Accepted | `"0x01"` works in Lotus but fails in go-ethereum |
| **Raw JSON Numbers** | Not accepted | Accepted (e.g., `42` without quotes) | Raw numbers work in Lotus but fail in go-ethereum |
| **Special Tags** | Handled directly in BlockNumber | Handled at higher level in EthBlockNumberOrHash | Tags must be handled correctly at the right level |
| **Error Handling** | Specific error types for each validation issue | Less specific errors | Error messages and handling differ |
| **Range Limits** | Rejects values > int64 max | uint64 range | Very large block numbers handled differently |
| **Example that fails in go-ethereum** | `"42"` (decimal without 0x) | | go-ethereum error: "hex number without 0x prefix" |
| **Example that fails in go-ethereum** | `"0x01"` (leading zero) | | go-ethereum error: "hex number with leading zero digits" |
| **Example that fails in go-ethereum** | `42` (raw JSON number) | | go-ethereum expects string format |

### Block Number or Hash Unmarshaling

| Feature | go-ethereum (BlockNumberOrHash) | Lotus (EthBlockNumberOrHash) | Compatibility Issues |
|---------|--------------------------------|------------------------------|----------------------|
| **JSON Structure** | ```{"blockNumber":"0x1"}``` or ```{"blockHash":"0x..."}``` | ```{"blockNumber":"0x1"}``` or ```{"blockHash":"0x..."}``` | Both require exact field name matching (case-sensitive) |
| **Special Tags** | Supports "earliest", "latest", "pending", "safe", "finalized" | Supports "earliest", "latest", "pending" at unmarshal level | "safe" and "finalized" not recognized at Lotus JSON unmarshaling level |
| **Internal Representation** | Special tags become negative integers | Special tags stored as string pointers | Different internal logic required |
| **Block Hash Detection** | 66 chars AND must have "0x" prefix | 66 chars AND must have "0x" prefix | Both implementations require "0x" prefix for hashes |
| **Tag Case Sensitivity** | Case sensitive (must be lowercase) | Case sensitive (must be lowercase) | Both reject "Latest" or "LATEST" |
| **Example that fails in both** | `{"BlockNumber":"0x1"}` (capital B) | | Neither accepts capitalized field names |
| **Example that fails in Lotus** | `"safe"` or `"finalized"` | | Not recognized at unmarshaling level in Lotus |
| **Example that fails in both** | Hash without "0x" prefix | | Both implementations require "0x" prefix for hashes |

### Block Tag Interpretation

| Block Tag | go-ethereum | Lotus | Compatibility Issues |
|-----------|-------------|-------|----------------------|
| **"earliest"** | Genesis block (block 0) | Genesis block (block 0) | Compatible |
| **"latest"** | Most recent mined block | Most recent mined block (parent of tipset) | Generally compatible |
| **"pending"** | Current state with pending transactions | Current state with pending messages | Semantic differences in what "pending" includes |
| **"safe"** | Safe head from consensus layer | 30 epochs behind "latest" | Different behavior, especially post-Merge |
| **"finalized"** | Finalized head from consensus layer | 900 epochs behind "latest" (Filecoin finality) | Different behavior, especially post-Merge |

### Key Implementation Differences:

1. **Type Representation**:
   - go-ethereum: Uses `BlockNumber` as int64 with negative constants for special blocks
   - Lotus: Uses `EthBlockNumberOrHash` with string pointers for special blocks

2. **Block Number Parsing**:
   - go-ethereum: Strict hex-only format with mandatory "0x" prefix and no leading zeros
   - Lotus: Flexible with both hex and decimal formats, optional "0x" prefix, accepts leading zeros

3. **Block Tags**:
   - Both support standard tags ("earliest", "latest", "pending")
   - Lotus implements "safe" and "finalized" as fixed offset from latest (30 and 900 epochs)
   - go-ethereum implements them according to Ethereum consensus rules

4. **Block Hash Validation**:
   - go-ethereum: Accepts 66-character strings, implicitly assumes "0x" prefix
   - Lotus: Requires both 66 characters AND explicit "0x" prefix

5. **JSON Field Names**:
   - go-ethereum: More forgiving with field name capitalization
   - Lotus: Strictly requires lowercase field names

6. **Internal Representation**:
   - Lotus has to translate between Filecoin's tipset-based chain and Ethereum's block-based chain
   - Results in subtly different behavior for methods that operate on block state

## Response Structure Differences

The table below details differences in how Lotus and go-ethereum format JSON-RPC responses:

| Response Type | go-ethereum | Lotus | Differences |
|---------------|-------------|-------|-------------|
| **Block Structure** | Complete Ethereum block with all fields | Simulated Ethereum block from Filecoin tipset | Missing or placeholder values for uncle-related fields, consensus fields |
| **Block Header Fields** | All Ethereum consensus fields present | Placeholder values for many fields | nonce=0, difficulty=0, mixHash empty, sha3Uncles is constant empty hash |
| **Transaction Structure** | Native Ethereum transaction types | Converted from Filecoin messages | Filecoin messages represented as EIP-1559 transactions regardless of original type |
| **Transaction Receipt** | Complete with status, logs, gasUsed, etc. | Simulated from Filecoin execution results | May have subtle differences in gas calculation and log format |
| **Post-Merge Fields** | Support for withdrawals, parentBeaconBlockRoot | Not supported | Missing fields in responses |
| **Post-Cancun Fields** | Support for blob gas fields | Not supported | Missing fields in responses |
| **Error Handling** | Ethereum-specific error codes | Similar error codes, but might differ for edge cases | Some errors may be reported differently |
| **State Access** | Direct Ethereum state access | Translated to Filecoin state tree operations | May have subtle differences in contract behavior |

### Key Compatibility Challenges:

1. **Architectural Differences**: Lotus maps Filecoin's fundamentally different architecture (tipsets vs. blocks, messages vs. transactions) to Ethereum's model.

2. **Address Translation**: Filecoin uses different address formats that must be mapped to Ethereum addresses.

3. **Transaction Type Support**: Lotus has limited support for newer Ethereum transaction types.

4. **Gas Model Differences**: Filecoin's gas model differs from Ethereum's, affecting calculations.

5. **Consensus Field Simulation**: Many Ethereum consensus fields are simulated with placeholder values in Lotus.

6. **Block Finality Interpretation**: Different interpretations of "latest", "safe", and "finalized" block tags.

7. **Missing Advanced Features**: No support for EIP-4844 blob transactions, access lists, and other advanced Ethereum features.

These differences may cause compatibility issues with certain Ethereum tools and applications, particularly those relying on newer Ethereum features or expecting specific consensus-related fields.
