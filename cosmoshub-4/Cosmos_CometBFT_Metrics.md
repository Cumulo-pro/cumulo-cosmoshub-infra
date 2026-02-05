# Cosmos Consensus System Metrics Documentation

## Introduction

This document provides a comprehensive reference for the consensus metrics used in the **Cosmos** project, which integrates the Ethereum Virtual Machine (EVM) with the XRP Ledger (Cosmos). These metrics, collected from the **CometBFT** consensus engine, are crucial for monitoring the performance, security, and health of validator nodes within the network.

The document is structured into different metric categories, including block synchronization, validator activity, transaction processing, memory and CPU usage, network performance, and mempool statistics. Each metric is explained with its description, value interpretation, and example values.

By leveraging these metrics, network operators and developers can ensure optimal performance, detect potential issues, and maintain a secure and efficient blockchain environment.

For ease of navigation, please refer to the **Table of Contents** below.

## Table of Contents

- [SYNC STATUS](#sync-status)
- [BLOCK GOSSIP STATUS](#block-gossip-status)
- [BLOCK SIZE](#block-size)
- [BYZANTINE VALIDATORS](#byzantine-validators)
- [MISSING VALIDATORS](#missing-validators)
- [CHAIN SIZE](#chain-size)
- [LATEST BLOCK HEIGHT](#latest-block-height)
- [TRANSACTIONS](#transactions)
- [DUPLICATE BLOCK PARTS](#duplicate-block-parts)
- [DUPLICATE VOTES](#duplicate-votes)
- [LATE VOTES](#late-votes)
- [PROPOSALS](#proposals)
- [ROUND VOTING POWER PERCENT](#round-voting-power-percent)
- [NUMBER OF ROUNDS](#number-of-rounds)
- [TOTAL TRANSACTIONS](#total-transactions)
- [VALIDATORS](#validators)
- [ACTIVE OUTBOUND CONNECTIONS](#active-outbound-connections)
- [MEMPOOL SIZE](#mempool-size)
- [MEMPOOL SIZE IN BYTES](#mempool-size-in-bytes)
- [NETWORK PEERS](#network-peers)
- [P2P MESSAGE RECEIVED BYTES](#p2p-message-received-bytes)
- [BLOCK PROCESSING TIME](#block-processing-time)
- [AVERAGE BLOCK PROCESSING TIME](#average-block-processing-time)
- [CONSENSUS PARAMETER UPDATES](#consensus-parameter-updates)
- [CPU USAGE](#cpu-usage)
- [FILE DESCRIPTORS](#file-descriptors)
- [NETWORK BYTES RECEIVED](#network-bytes-received)
- [NETWORK BYTES TRANSMITTED](#network-bytes-transmitted)
- [MEMORY USAGE](#memory-usage)
- [PROCESS START TIME](#process-start-time)
- [HOW TO USE THESE METRICS](#how-to-use-these-metrics)
- [EXAMPLE PROMETHEUS OUTPUT](#example-prometheus-output)

_______________________________________

## SYNC STATUS
### Metric: `cometbft_blocksync_syncing`

**Description:**  
A metric that indicates whether a node in the Cosmos consensus system is syncing blocks. When this metric has a value of **1**, it means that the node is actively syncing blocks. Conversely, a value of **0** indicates that the node is not currently syncing blocks.

**Value Interpretation:**

| Value | Status |
|--------|--------------|
| `1`   | SYNC (syncing blocks) |
| `0`   | NO SYNC (not syncing blocks) |

---

## BLOCK GOSSIP STATUS
### Metric: `cometbft_consensus_block_gossip_parts_received`

**Description:**  
This metric represents the number of block parts received by a node. These parts are categorized based on whether they belong to the block the node is currently trying to assemble.

**Value Interpretation:**  

| Labels | Description |
|------------------------|----------------------------------------------|
| `matches_current="true"`  | Number of block parts received that belong to the block the node is assembling. |
| `matches_current="false"` | Number of block parts received that do not belong to the block the node is assembling. |

---

## BLOCK SIZE
### Metric: `cometbft_consensus_block_size_bytes`

**Description:**  
Indicates the size of the latest block in bytes.

**Example Value:**  
- `4929` bytes (for `chain_id="cosmoshub-4"`)

---

## BYZANTINE VALIDATORS
### Metric: `cometbft_consensus_byzantine_validators`

**Description:**  
Represents the number of validators that attempted to double sign.

**Example Value:**  
- `0` (no byzantine validators detected)

### Metric: `cometbft_consensus_byzantine_validators_power`

**Description:**  
Indicates the total power of the byzantine validators.

**Example Value:**  
- `0` (no power attributed to byzantine validators)

---

## CHAIN HEIGHT
### Metric: `cometbft_consensus_height`

**Description:**  
Indicates the current height of the blockchain.

**Example Value:**  
- `14413294` blocks (for `chain_id="cosmoshub-4"`).

---

## MISSING VALIDATORS
### Metric: `cometbft_consensus_missing_validators`

**Description:**  
Represents the number of validators who failed to sign.

**Example Value:**  
- `1` validator missed signing.

### Metric: `cometbft_consensus_missing_validators_power`

**Description:**  
Indicates the total power of the validators who missed signing.

**Example Value:**  
- `1` (total voting power of missing validators)

---

## MISSING VALIDATORS POWER
### Metric: `cometbft_consensus_missing_validators_power`

**Description:**  
Indicates the **total voting power** of the validators who failed to sign a block. This metric is crucial for identifying potential issues with validator participation and network security.

A nonzero value suggests that some validators did not sign a block, which may indicate:
- **Temporary network issues** affecting validator connectivity.
- **Validator downtime** due to maintenance or failures.
- **Potential malicious behavior**, if a validator is intentionally avoiding signing.

**Example Value:**  
- `1` total voting power of missing validators.

**Interpretation:**  
- **A value of `0`** means all validators signed as expected.
- **A nonzero value** indicates missing validator signatures, which may impact consensus security.
- **Persistent high values** may require investigation into validator performance and network conditions.

---

## CHAIN SIZE
### Metric: `cometbft_consensus_chain_size_bytes`

**Description:**  
Represents the total size of the blockchain in bytes.

**Example Value:**  
- `77858` bytes (for `chain_id="cosmoshub-4"`)


## LATEST BLOCK HEIGHT
### Metric: `cometbft_consensus_latest_block_height`

**Description:**  
Indicates the latest confirmed block height in the blockchain.

**Example Value:**  
- `14413293` blocks (for `chain_id="cosmoshub-4"`)

### Metric: `cometbft_consensus_height`

**Description:**  
Represents the current height of the blockchain.

**Example Value:**  
- `14413294` blocks (for `chain_id="cosmoshub-4"`)

---

## TRANSACTIONS
### Metric: `cometbft_consensus_num_txs`

**Description:**  
Indicates the number of transactions included in the latest block.

**Example Value:**  
- `1` transaction in the last block.

---

## DUPLICATE BLOCK PARTS
### Metric: `cometbft_consensus_duplicate_block_part`

**Description:**  
Indicates the number of times the node has received a **duplicate block part**. This metric helps identify inefficiencies in block propagation and potential redundancy in network communication.

**Example Value:**  
- `34` duplicate block parts detected.

**Interpretation:**  
- **A low value** suggests efficient block propagation.
- **A high value** may indicate network congestion, improper peer behavior, or redundant data transmission.
- **Persistent high values** may require network optimization.

---

## DUPLICATE VOTES
### Metric: `cometbft_consensus_duplicate_vote`

**Description:**  
Represents the number of times the node has received a **duplicate vote**. This can occur due to network issues, validator misconfigurations, or attempts at Byzantine behavior.

**Example Value:**  
- `3022` duplicate votes received.

**Interpretation:**  
- **A few duplicate votes** may be normal due to natural network conditions.
- **A high number of duplicate votes** might indicate poor validator communication or network instability.
- **Persistent high values** could suggest a need for validator synchronization improvements.

---

## LATE VOTES
### Metric: `cometbft_consensus_late_votes`

**Description:**  
Stores the number of votes received by this node that correspond to earlier heights and rounds than the node is currently in.

**Value Interpretation:**  

| Vote Type | Description |
|-----------------|-------------------------------------------|
| `precommit` | Late precommit votes received. |
| `prevote`   | Late prevote votes received. |

**Example Values:**  

| `chain_id="cosmoshub-4"` | `vote_type` | Value |
|----------------------|-------------|--------|
| cosmoshub-4 | `precommit` | `1752` |
| cosmoshub-4 | `prevote` | `149` |

---

## PROPOSALS
### Metric: `cometbft_consensus_proposal_receive_count`

**Description:**  
Represents the total number of proposals received by this node since process start. The metric is annotated by the status of the proposal from the application, either **accepted** or **rejected**.

**Example Value:**  
- `18` proposals accepted.

## ROUND VOTING POWER PERCENT
### Metric: `cometbft_consensus_round_voting_power_percent`

**Description:**  
Represents the percentage of the total voting power received in a round. The value starts at `0` for each round and increases towards `1.0` as additional voting power is observed.  

**Value Interpretation:**  

| Vote Type | Description |
|-----------------|-------------------------------------------|
| `precommit` | Percentage of voting power received in precommit phase. |
| `prevote`   | Percentage of voting power received in prevote phase. |

**Example Values:**  

| `chain_id="cosmoshub-4"` | `vote_type` | Value |
|----------------------|-------------|--------|
| cosmoshub-4 | `precommit` | `2.03%` |
| cosmoshub-4 | `prevote` | `2.79%` |

---

## NUMBER OF ROUNDS
### Metric: `cometbft_consensus_rounds`

**Description:**  
Indicates the number of rounds the consensus process has gone through.

**Example Value:**  
- `1` round completed.

## TOTAL TRANSACTIONS
### Metric: `cometbft_consensus_total_txs`

**Description:**  
Indicates the total number of transactions processed in the blockchain.

**Example Value:**  
- `2` total transactions recorded.

---

## VALIDATORS
### Metric: `cometbft_consensus_validators`

**Description:**  
Represents the total number of validators participating in the consensus process.

**Example Value:**  
- `34` validators.

### Metric: `cometbft_consensus_validators_power`

**Description:**  
Indicates the total voting power of all validators combined.

**Example Value:**  
- `34` total voting power.

---

## ACTIVE OUTBOUND CONNECTIONS
### Metric: `cometbft_mempool_active_outbound_connections`

**Description:**  
Represents the number of active outbound connections used for gossiping transactions. This metric helps track how many connections are actively involved in transaction propagation. It is an **experimental feature**.

**Example Value:**  
- `7` active outbound connections.

---

## MEMPOOL SIZE
### Metric: `cometbft_mempool_size`

**Description:**  
Indicates the number of uncommitted transactions currently in the mempool. The mempool stores transactions that have been broadcasted but not yet included in a block.

**Example Value:**  
- `0` transactions in mempool.

---

## MEMPOOL SIZE IN BYTES
### Metric: `cometbft_mempool_size_bytes`

**Description:**  
Represents the total size of the mempool in bytes. This metric gives an estimate of the memory footprint of the pending transactions waiting to be confirmed.

**Example Value:**  
- `0` bytes in mempool.

---

## NETWORK PEERS
### Metric: `cometbft_p2p_peers`

**Description:**  
Indicates the number of peers currently connected to the node.

**Example Value:**  
- `7` peers connected.

---

## P2P MESSAGE RECEIVED BYTES
### Metric: `cometbft_p2p_message_receive_bytes_total`

**Description:**  
Represents the **total number of bytes received** for each message type over the peer-to-peer (P2P) network. This metric helps monitor network activity, assess bandwidth usage, and detect anomalies in message propagation.

**Example Values:**  

| `chain_id="cosmoshub-4"` | Message Type | Bytes Received |
|----------------------|------------------------------|---------------|
| cosmoshub-4 | `blocksync_StatusResponse` | `72` |
| cosmoshub-4 | `consensus_BlockPart` | `223082` |
| cosmoshub-4 | `consensus_HasVote` | `41120` |
| cosmoshub-4 | `consensus_NewRoundStep` | `3723` |
| cosmoshub-4 | `consensus_NewValidBlock` | `2716` |
| cosmoshub-4 | `consensus_Proposal` | `8900` |
| cosmoshub-4 | `consensus_Vote` | `692075` |
| cosmoshub-4 | `consensus_VoteSetBits` | `332` |
| cosmoshub-4 | `consensus_VoteSetMaj23` | `742` |
| cosmoshub-4 | `mempool_Txs` | `8286` |
| cosmoshub-4 | `p2p_PexAddrs` | `14157` |

**Interpretation:**  
- **High values in `consensus_BlockPart`** indicate significant data transfer during block propagation.
- **A large number of `consensus_Vote` messages** suggests heavy consensus participation.
- **`mempool_Txs` traffic** helps track transaction gossiping across nodes.
- **Unusually high values in any category** may indicate inefficient message handling or potential DDoS attempts.


---

## BLOCK PROCESSING TIME
### Metric: `cometbft_state_block_processing_time`

**Description:**  
Represents the time spent processing `FinalizeBlock`. This metric is recorded as a histogram, showing the number of blocks processed within different time ranges.

**Value Interpretation:**  

| Processing Time (seconds) | Number of Blocks Processed |
|--------------------------|----------------------------|
| `≤ 1`   | 0  |
| `≤ 11`  | 1  |
| `≤ 21`  | 13 |
| `≤ 31`  | 18 |
| `≤ 41`  | 19 |
| `≤ 51`  | 19 |
| `≤ 61`  | 19 |
| `≤ 71`  | 19 |
| `≤ 81`  | 19 |
| `≤ 91`  | 19 |
| `+Inf`  | 19 |

**Additional Metrics:**
- **Total Processing Time:** `391.85` seconds.
- **Total Blocks Processed:** `19`.

---

## AVERAGE BLOCK PROCESSING TIME
### Metric: `cometbft_state_block_processing_time`

**Description:**  
This metric provides the **average time** it takes for the node to process a block. It is calculated by dividing the **total block processing time** (`cometbft_state_block_processing_time_sum`) by the **total number of blocks processed** (`cometbft_state_block_processing_time_count`).

Monitoring the **average block processing time** is crucial for evaluating the efficiency of block finalization. If the average time increases, it may indicate **performance bottlenecks** or issues with the node's processing power. Keeping this metric within an optimal range helps ensure the **smooth functioning of the network** and avoids delays in transaction finalization.

**Calculation:**  
\[
\text{Average Block Processing Time} = \frac{\text{cometbft_state_block_processing_time_sum}}{\text{cometbft_state_block_processing_time_count}}
\]

**Example Values:**  

| Metric | Value |
|----------------------|--------------|
| `cometbft_state_block_processing_time_sum` | `391.85` seconds |
| `cometbft_state_block_processing_time_count` | `19` blocks |
| **Average Block Processing Time** | `20.62` seconds per block |

**Interpretation:**  
- **Lower values** indicate efficient block processing.
- **Higher values** suggest performance issues or network congestion.
- **Sudden spikes** may require investigation into node performance and resource allocation.


---

## CONSENSUS PARAMETER UPDATES
### Metric: `cometbft_state_consensus_param_updates`

**Description:**  
Represents the number of consensus parameter updates returned by the application since the process started.

**Example Value:**  
- `19` updates.

## CPU USAGE
### Metric: `process_cpu_seconds_total`

**Description:**  
Represents the total user and system CPU time spent by the process in seconds.

**Example Value:**  
- `25.81` seconds.

---

## FILE DESCRIPTORS
### Metric: `process_max_fds`

**Description:**  
Indicates the maximum number of file descriptors that the process can open.

**Example Value:**  
- `65535` file descriptors.

### Metric: `process_open_fds`

**Description:**  
Represents the number of file descriptors currently open by the process.

**Example Value:**  
- `119` open file descriptors.

---

## NETWORK BYTES RECEIVED
### Metric: `process_network_receive_bytes_total`

**Description:**  
Indicates the **total number of bytes received** by the process over the network. This metric helps track inbound network traffic and assess data consumption.

**Example Value:**  
- `8.88 TB` (`8.888071873671e+12` bytes).

**Interpretation:**  
- **Higher values** indicate significant inbound data transfer.
- **Sudden spikes** may suggest increased network activity or potential traffic surges.
- **Consistently low values** could indicate connectivity issues or underutilization of the node.

---

## NETWORK BYTES TRANSMITTED
### Metric: `process_network_transmit_bytes_total`

**Description:**  
Indicates the **total number of bytes sent** by the process over the network. This metric helps monitor outbound traffic and detect anomalies in data transmission.

**Example Value:**  
- `2.83 TB` (`2.83214477635e+12` bytes).

**Interpretation:**  
- **High values** suggest substantial outbound communication, which may be normal for validators or full nodes.
- **Sudden increases** may indicate excessive message broadcasting or potential DDoS activity.
- **Very low values** might point to inefficient node participation in the network.

---

## MEMORY USAGE
### Metric: `process_resident_memory_bytes`

**Description:**  
Represents the resident memory size (RAM) currently used by the process in bytes.

**Example Value:**  
- `294.38 MB` (`2.94375424e+08` bytes).

### Metric: `process_virtual_memory_bytes`

**Description:**  
Indicates the total virtual memory size used by the process in bytes.

**Example Value:**  
- `3.73 GB` (`3.730767872e+09` bytes).

### Metric: `process_virtual_memory_max_bytes`

**Description:**  
Indicates the maximum amount of virtual memory available to the process in bytes.

**Example Value:**  
- `18.44 EB` (`1.8446744073709552e+19` bytes).

---

## PROCESS START TIME
### Metric: `process_start_time_seconds`

**Description:**  
Represents the start time of the process since the Unix epoch, in seconds.

**Example Value:**  
- `1.73901610886e+09` (approximately `2025-02-08 00:21:48 UTC`).








---

## How to Use These Metrics
- **Monitor synchronization status:** Use `cometbft_blocksync_syncing` to check if a node is syncing.
- **Check block size trends:** Analyze `cometbft_consensus_block_size_bytes` to monitor storage growth.
- **Detect byzantine behavior:** Use `cometbft_consensus_byzantine_validators` and `cometbft_consensus_byzantine_validators_power` to check for malicious validators.
- **Observe blockchain growth:** Monitor `cometbft_consensus_chain_size_bytes`, `cometbft_consensus_height`, and `cometbft_consensus_latest_block_height`.
- **Identify network inefficiencies:** Track `cometbft_consensus_duplicate_block_part` and `cometbft_consensus_duplicate_vote` to detect excessive duplicate messages.
- **Analyze validator participation:** Use `cometbft_consensus_missing_validators` and `cometbft_consensus_missing_validators_power` to detect absent validators.
- **Evaluate proposal processing:** `cometbft_consensus_proposal_receive_count` can indicate how well the network handles proposals.

---

### Example Prometheus Output
```plaintext
# HELP cometbft_consensus_late_votes LateVotes stores the number of votes that were received by this node that correspond to earlier heights and rounds than this node is currently in.
# TYPE cometbft_consensus_late_votes counter
cometbft_consensus_late_votes{chain_id="cosmoshub-4",vote_type="precommit"} 1752
cometbft_consensus_late_votes{chain_id="cosmoshub-4",vote_type="prevote"} 149

# HELP cometbft_consensus_latest_block_height The latest block height.
# TYPE cometbft_consensus_latest_block_height gauge
cometbft_consensus_latest_block_height{chain_id="cosmoshub-4"} 1.4413293e+07

# HELP cometbft_consensus_proposal_receive_count ProposalReceiveCount is the total number of proposals received by this node since process start.
# TYPE cometbft_consensus_proposal_receive_count counter
cometbft_consensus_proposal_receive_count{chain_id="cosmoshub-4",status="accepted"} 18
