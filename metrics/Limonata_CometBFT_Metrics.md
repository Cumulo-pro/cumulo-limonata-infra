# Limonata CometBFT Metrics Documentation

## Introduction

This document provides a comprehensive reference for the metrics used in **Limonata**, an EVM-compatible chain built on **CometBFT** (via the `evmd` binary, a Cosmos EVM example app). Unlike Gnoland (TM2), which requires an OpenTelemetry Collector to bridge metrics into Prometheus, Limonata's `evmd` exposes Prometheus metrics **natively** through CometBFT's built-in instrumentation. No collector is required, Prometheus scrapes the node's `/metrics` endpoint directly.

The metrics are organized into categories covering consensus/block production, validator signing, P2P network health, mempool activity, and ABCI application performance. Each metric is explained with its description, value interpretation, Prometheus query examples, and real values observed on the **Limonata Testnet** by **Cumulo**.

For ease of navigation, refer to the Table of Contents below.

> **Infrastructure:** Metrics collected from the Cumulo validator node on Limonata Testnet (`limonata_10777-1`), exposed natively via CometBFT Prometheus instrumentation and scraped directly by Prometheus. Dashboard available at [Cumulo Grafana](https://cumulo.pro/services/).

---

## Table of Contents

- [BLOCK HEIGHT](#block-height)
- [BLOCK INTERVAL](#block-interval)
- [CONSENSUS ROUNDS](#consensus-rounds)
- [TOTAL TRANSACTIONS](#total-transactions)
- [TRANSACTIONS PER BLOCK](#transactions-per-block)
- [VALIDATOR VOTING POWER](#validator-voting-power)
- [MISSED BLOCKS](#missed-blocks)
- [SIGNING LAG](#signing-lag)
- [BYZANTINE VALIDATORS](#byzantine-validators)
- [P2P PEERS](#p2p-peers)
- [P2P BANDWIDTH](#p2p-bandwidth)
- [MEMPOOL REAPED TRANSACTIONS](#mempool-reaped-transactions)
- [MEMPOOL DUPLICATE RECEPTIONS](#mempool-duplicate-receptions)
- [MEMPOOL TRANSACTION SIZE](#mempool-transaction-size)
- [MEMPOOL BATCH SIZE](#mempool-batch-size)
- [ABCI METHOD LATENCY](#abci-method-latency)
- [HOW TO USE THESE METRICS](#how-to-use-these-metrics)
- [EXAMPLE PROMETHEUS OUTPUT](#example-prometheus-output)

---

## BLOCK HEIGHT

### Metric: `cometbft_consensus_height`

**Description:**
A gauge reporting the current block height the node has reached. The primary indicator of sync progress.

**Prometheus Query Examples:**

```
# Current block height
cometbft_consensus_height{job="limonata_validator"}

# Sync speed (blocks per minute)
rate(cometbft_consensus_height{job="limonata_validator"}[5m]) * 60
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric      | Value    |
| ----------- | -------- |
| Height      | 776,000  |
| Sync speed  | ~18.5 blocks/min |

**Interpretation:**

- A steadily increasing height confirms the node is actively processing blocks
- Sync speed dropping toward 0 while `catching_up` is still true in `/status` indicates a stalled replay
- Compare against `rpc-testnet.limonata.cumulo.me` to confirm the node has reached the network tip

---

## BLOCK INTERVAL

### Metric: `cometbft_consensus_block_interval_seconds`

**Description:**
A histogram measuring the time elapsed between consecutive blocks. Reflects how quickly the network reaches agreement and finalizes blocks.

**Value Interpretation:**

| Percentile | Description                                                          |
| ---------- | --------------------------------------------------------------------- |
| `p50`      | Median block time, typical interval under normal conditions           |
| `p90`      | 90th percentile, covers most blocks including slight delays           |
| `p99`      | 99th percentile, worst-case block times, useful for detecting stalls  |

**Prometheus Query Examples:**

```
# Median block interval
histogram_quantile(0.5, sum(rate(cometbft_consensus_block_interval_seconds_bucket{job="limonata_validator"}[10m])) by (le))

# p99 block interval
histogram_quantile(0.99, sum(rate(cometbft_consensus_block_interval_seconds_bucket{job="limonata_validator"}[10m])) by (le))
```

**Example Values (Limonata Testnet, Cumulo):**

| Percentile | Mean    | Max     | Min     |
| ---------- | ------- | ------- | ------- |
| p50        | 2.08 s  | 2.10 s  | 1.88 s  |
| p90        | 8.36 s  | 8.45 s  | 6.72 s  |
| p99        | 9.84 s  | 9.93 s  | 9.67 s  |

**Interpretation:**

- The large gap between p50 and p90/p99 during replay is expected, historical blocks are read from disk far faster than they were originally produced, so most intervals are small (p50) with periodic larger gaps at checkpoint/commit boundaries (p90/p99)
- Once fully synced, p50/p90/p99 should converge close to the chain's configured block time
- Sustained p99 far above the configured block time after sync completes warrants checking peer connectivity and consensus rounds

---

## CONSENSUS ROUNDS

### Metric: `cometbft_consensus_rounds`

**Description:**
A gauge/histogram tracking the number of consensus rounds required to finalize each block. A round above 0 means the first proposer attempt failed and the network had to retry.

**Prometheus Query Examples:**

```
# Average rounds per block
cometbft_consensus_rounds{job="limonata_validator"}
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric | Mean  | Max |
| ------ | ----- | --- |
| Rounds | 0.250 | 1   |

**Interpretation:**

- **Mean close to 0** indicates the network almost always finalizes on the first round, healthy behavior
- **Rising mean or frequent max values > 1** suggests proposer timeouts, network latency, or an unresponsive validator in the active set
- Correlate spikes with `cometbft_p2p_peers` drops to rule out connectivity issues

---

## TOTAL TRANSACTIONS

### Metric: `cometbft_consensus_total_txs`

**Description:**
A counter tracking the cumulative number of transactions processed by the chain since genesis.

**Prometheus Query Examples:**

```
# Total transactions
cometbft_consensus_total_txs{job="limonata_validator"}

# Transaction growth rate
rate(cometbft_consensus_total_txs{job="limonata_validator"}[1h])
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric      | Value |
| ----------- | ----- |
| Total txs   | 41    |

**Interpretation:**

- Low cumulative totals are expected and normal on an early-stage testnet
- A flat total_txs value alongside an increasing block height simply reflects low network activity, not a fault

---

## TRANSACTIONS PER BLOCK

### Metric: `cometbft_consensus_num_txs`

**Description:**
A gauge reporting the number of transactions included in the most recently processed block.

**Prometheus Query Examples:**

```
# Transactions in latest block
cometbft_consensus_num_txs{job="limonata_validator"}
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric        | Mean    | Max | Total |
| ------------- | ------- | --- | ----- |
| Txs per block | 0.00465 | 1   | 3     |

**Interpretation:**

- **Values near 0** are expected on a lightly loaded testnet, most blocks are empty
- Sustained non-zero values would indicate the start of meaningful transaction throughput

---

## VALIDATOR VOTING POWER

### Metrics: `cometbft_consensus_validator_power`, `cometbft_consensus_validators_power`

**Description:**
`validator_power` reports this node's own voting power. `validators_power` reports the total voting power of the active validator set. Together they express this validator's relative share.

**Prometheus Query Examples:**

```
# This validator's voting power
cometbft_consensus_validator_power{job="limonata_validator"}

# Total network voting power
cometbft_consensus_validators_power{job="limonata_validator"}

# Voting power share (%)
100 * sum(cometbft_consensus_validator_power{job="limonata_validator"}) by (instance)
    / sum(cometbft_consensus_validators_power{job="limonata_validator"}) by (instance)
```

> Use `sum(...) by (instance)` on both sides before dividing. Without it, Prometheus vector matching can silently return no data if either metric carries extra labels that don't line up one-to-one.

**Example Values (Limonata Testnet, Cumulo):**

| Metric              | Value   |
| -------------------- | ------- |
| Validator power       | 1,810   |
| Network total power   | 144,000 |
| Share                 | 1.25%   |

**Interpretation:**

- A stable share over time indicates no re-delegation or slashing events
- A sudden drop in share without a corresponding drop in absolute power indicates other validators are gaining stake, not that this validator lost any
- A drop in absolute power together with share is a signal to check for slashing, jailing, or an unintended unbond

---

## MISSED BLOCKS

### Metric: `cometbft_consensus_validator_missed_blocks`

**Description:**
A **cumulative counter** of blocks where this validator's signature was expected but absent from the commit, counted since the `evmd` process started.

**Critical caveat:** this counter is cumulative from process start, not "currently missing." If the node performed a historical replay or state-sync from a snapshot predating when the validator was bonded, every one of those pre-bonding blocks is counted as "missed" even though the validator was never expected to sign them. The raw value can look alarmingly high on a validator that is, in reality, signing perfectly.

**Prometheus Query Examples:**

```
# WRONG for live health checks - includes historical replay noise:
cometbft_consensus_validator_missed_blocks{job="limonata_validator"}

# CORRECT - only counts blocks missed in the recent window:
increase(cometbft_consensus_validator_missed_blocks{job="limonata_validator"}[10m])
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric                                  | Value  |
| ---------------------------------------- | ------ |
| Raw cumulative counter                    | 8,630  |
| `increase(...[10m])` (during replay)      | 184    |
| `increase(...[10m])` (fully synced)       | 0      |

**Interpretation:**

- The raw counter (8,630) and the 10-minute increase (184, matching the ~18.5 blocks/min replay speed almost exactly) together confirmed this validator was still catching up historically, not actually failing to sign live blocks
- Cross-check against the chain explorer's uptime/signing page (e.g. `testnet.limonata.valopers.com`), which reflects real signing history and is the source of truth
- Only alert on `increase(...[Nm]) > 0` once the node is confirmed fully synced (`catching_up: false` in `/status`)

---

## SIGNING LAG

### Metric: `cometbft_consensus_validator_last_signed_height`

**Description:**
A gauge reporting the last block height at which this validator successfully signed. Compared against current height, it measures how far behind (if at all) the validator's signature history is.

**Prometheus Query Examples:**

```
# Signing lag (current height - last signed height)
sum(cometbft_consensus_height{job="limonata_validator"}) by (instance)
  - sum(cometbft_consensus_validator_last_signed_height{job="limonata_validator"}) by (instance)
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric | Max | Last |
| ------ | --- | ---- |
| Lag    | 380 | 66   |

**Interpretation:**

- During replay, lag reflects the distance to the point where the validator became bonded, not a live signing problem
- Once fully synced, lag should stay at or near 0
- A growing lag on a fully synced node is a real signing problem: check `priv_validator_state.json`, key permissions, and peer connectivity immediately

---

## BYZANTINE VALIDATORS

### Metric: `cometbft_consensus_byzantine_validators`

**Description:**
A gauge reporting the number of validators found to have submitted conflicting (double-signed) votes in the most recent commit.

**Prometheus Query Examples:**

```
cometbft_consensus_byzantine_validators{job="limonata_validator"}
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric               | Value |
| --------------------- | ----- |
| Byzantine validators   | 0     |

**Interpretation:**

- **0** is the expected, healthy value at all times
- Any value above 0 indicates evidence of double-signing was included in a block and should be investigated and reported immediately, this affects network security and can trigger slashing

---

## P2P PEERS

### Metric: `cometbft_p2p_peers`

**Description:**
A gauge reporting the current number of connected P2P peers (inbound and outbound combined).

**Prometheus Query Examples:**

```
cometbft_p2p_peers{job="limonata_validator"}
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric | Mean | Min | Max | Last |
| ------ | ---- | --- | --- | ---- |
| Peers  | 9.98 | 7   | 10  | 10   |

**Interpretation:**

- Consistently at or near the configured peer limit is healthy
- A drop toward 0 indicates a P2P/firewall/seed-node issue and directly threatens the ability to receive new blocks and gossip votes
- Occasional dips (e.g. to 7) are normal peer churn and not cause for alarm on their own

---

## P2P BANDWIDTH

### Metrics: `cometbft_p2p_peer_receive_bytes_total`, `cometbft_p2p_peer_send_bytes_total`

**Description:**
Counters tracking total bytes received from and sent to peers, broken down by message type internally.

**Prometheus Query Examples:**

```
# Receive rate (bytes/s)
sum(rate(cometbft_p2p_peer_receive_bytes_total{job="limonata_validator"}[5m])) by (instance)

# Send rate (bytes/s)
sum(rate(cometbft_p2p_peer_send_bytes_total{job="limonata_validator"}[5m])) by (instance)
```

**Example Values (Limonata Testnet, Cumulo):**

| Direction | Mean       | Max        |
| --------- | ---------- | ---------- |
| Receive   | 23.7 kB/s  | 29.2 kB/s  |
| Send      | 18.8 kB/s  | 22.3 kB/s  |

**Interpretation:**

- Bandwidth scales with peer count and block/vote gossip volume, expect it to rise with more peers or higher chain activity
- A sudden drop to near-zero on both directions while peers remain connected can indicate the node has stopped participating in gossip despite an open connection

---

## MEMPOOL REAPED TRANSACTIONS

### Metric: `cometbft_mempool_reaped_txs`

**Description:**
A counter tracking the total number of transactions reaped (pulled) from the mempool for inclusion in a block since process start.

> This build does not expose a live "current mempool size" gauge. `reaped_txs` and its rate are the closest available proxy for mempool throughput.

**Prometheus Query Examples:**

```
# Total reaped
cometbft_mempool_reaped_txs{job="limonata_validator"}

# Reap rate (throughput)
sum(rate(cometbft_mempool_reaped_txs{job="limonata_validator"}[10m])) by (instance)
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric       | Value |
| ------------- | ----- |
| Total reaped   | 39-40 |

**Interpretation:**

- Matches the low `total_txs` figure, consistent with a lightly used testnet
- Sustained non-zero reap rate under real load would confirm the mempool is being drained as expected rather than backing up

---

## MEMPOOL DUPLICATE RECEPTIONS

### Metric: `cometbft_mempool_already_received_txs`

**Description:**
A counter tracking how many times the node received a transaction it had already seen, a normal side effect of gossip redundancy across multiple peers.

**Prometheus Query Examples:**

```
sum(rate(cometbft_mempool_already_received_txs{job="limonata_validator"}[10m])) by (instance)
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric              | Value |
| --------------------- | ----- |
| Already received (raw) | 333   |

**Interpretation:**

- Some duplication is expected and healthy, it reflects redundant gossip paths reaching the node
- An excessively high ratio of duplicates to reaped txs may indicate an oversized peer set relative to actual chain activity, worth reviewing but rarely urgent

---

## MEMPOOL TRANSACTION SIZE

### Metric: `cometbft_mempool_tx_size_bytes`

**Description:**
A histogram of transaction sizes, in bytes, as they enter the mempool.

**Prometheus Query Examples:**

```
# Median tx size
histogram_quantile(0.5, sum(rate(cometbft_mempool_tx_size_bytes_bucket{job="limonata_validator"}[30m])) by (le))

# p99 tx size
histogram_quantile(0.99, sum(rate(cometbft_mempool_tx_size_bytes_bucket{job="limonata_validator"}[30m])) by (le))
```

**Example Values (Limonata Testnet, Cumulo):**

| Metric        | Value      |
| -------------- | ---------- |
| Sum / count     | 12,155 bytes / 40 txs |
| Average size    | ~304 bytes |

**Interpretation:**

- Typical EVM transfer/call sizes fall in the low hundreds of bytes, consistent with observed values
- A rising p99 over time may indicate larger contract calls or batched operations becoming more common

---

## MEMPOOL BATCH SIZE

### Metric: `cometbft_mempool_batch_size`

**Description:**
A histogram of how many transactions are grouped per inbound/outbound gossip batch, labeled by `dir`.

**Prometheus Query Examples:**

```
# Median inbound batch size
histogram_quantile(0.5, sum(rate(cometbft_mempool_batch_size_bucket{job="limonata_validator", dir="inbound"}[30m])) by (le))

# Median outbound batch size
histogram_quantile(0.5, sum(rate(cometbft_mempool_batch_size_bucket{job="limonata_validator", dir="outbound"}[30m])) by (le))
```

**Example Values (Limonata Testnet, Cumulo):**

| Direction | Count |
| --------- | ----- |
| Inbound   | 373   |
| Outbound  | 39    |

**Interpretation:**

- Inbound batches far outnumbering outbound is expected, the node receives gossip from many peers but only relays its own reaps outward
- Mostly single-tx batches (`le="1"` dominating the histogram) is typical at low transaction volume

---

## ABCI METHOD LATENCY

### Metric: `cometbft_abci_connection_method_timing_seconds`

**Description:**
A histogram measuring how long each ABCI method call takes between CometBFT and the application (`evmd`), labeled by `method`. This is the clearest signal of application-level performance bottlenecks, as opposed to consensus/network delays.

**Prometheus Query Examples:**

```
# p99 latency by method
histogram_quantile(0.99, sum(rate(cometbft_abci_connection_method_timing_seconds_bucket{job="limonata_validator"}[10m])) by (le, method))
```

**Example Values (Limonata Testnet, Cumulo):**

| Method             | Mean      | Max       |
| ------------------- | --------- | --------- |
| `commit`             | 95.3 ms   | 97.3 ms   |
| `finalize_block`     | 21.8 ms   | 64.0 ms   |
| `flush`              | 99.0 µs   | 99.0 µs   |
| `insert_tx`          | 674 µs    | 1.98 ms   |
| `prepare_proposal`   | 9.46 ms   | 19.9 ms   |
| `process_proposal`   | 2.03 ms   | 7.74 ms   |
| `reap_txs`           | 99.0 µs   | 99.1 µs   |

**Interpretation:**

- `commit` is consistently the most expensive method, expected, it includes state persistence to disk
- `finalize_block` max spikes (64 ms vs 21.8 ms mean) are worth watching if they grow, they directly extend block interval
- A sustained rise in any method's p99 points to an application-level bottleneck (disk I/O, state size growth, or CPU contention) rather than a networking or consensus issue

---

## How to Use These Metrics

- **Monitor node sync status:** Track `cometbft_consensus_height` growth rate. A rate dropping to 0 while `catching_up: true` means the node has stalled.
- **Detect real signing problems, not replay noise:** Always use `increase(cometbft_consensus_validator_missed_blocks[Nm])`, never the raw cumulative counter, and only alert on it once the node is confirmed fully synced.
- **Track network health:** Monitor `cometbft_p2p_peers`, a sustained drop threatens both block reception and vote gossip.
- **Evaluate consensus health:** Use `cometbft_consensus_rounds` and `cometbft_consensus_block_interval_seconds` p90/p99 together to distinguish network latency from application slowness.
- **Isolate application bottlenecks:** Use `cometbft_abci_connection_method_timing_seconds` by method, rising `commit` or `finalize_block` latency points at the application/disk layer, not the network.
- **Watch decentralization:** Track `cometbft_consensus_validator_power` / `cometbft_consensus_validators_power` share over time.
- **Security:** `cometbft_consensus_byzantine_validators` should always read 0, alert immediately on any nonzero value.

---

## Example Prometheus Output

```
# HELP cometbft_consensus_height Height of the chain
# TYPE cometbft_consensus_height gauge
cometbft_consensus_height{chain_id="limonata_10777-1",instance="54.39.128.229:27660",job="limonata_validator"} 776412

# HELP cometbft_consensus_validator_missed_blocks Total missed blocks since process start
# TYPE cometbft_consensus_validator_missed_blocks counter
cometbft_consensus_validator_missed_blocks{chain_id="limonata_10777-1",instance="54.39.128.229:27660",job="limonata_validator"} 8630

# HELP cometbft_consensus_byzantine_validators Number of validators who tried to double sign
# TYPE cometbft_consensus_byzantine_validators gauge
cometbft_consensus_byzantine_validators{chain_id="limonata_10777-1",instance="54.39.128.229:27660",job="limonata_validator"} 0

# HELP cometbft_p2p_peers Number of peers
# TYPE cometbft_p2p_peers gauge
cometbft_p2p_peers{chain_id="limonata_10777-1",instance="54.39.128.229:27660",job="limonata_validator"} 10

# HELP cometbft_mempool_already_received_txs Number of duplicate transaction reception
# TYPE cometbft_mempool_already_received_txs counter
cometbft_mempool_already_received_txs{chain_id="limonata_10777-1",instance="54.39.128.229:27660",job="limonata_validator"} 333

# HELP cometbft_mempool_reaped_txs ReapedTxs is the number of transactions reaped from the mempool
# TYPE cometbft_mempool_reaped_txs counter
cometbft_mempool_reaped_txs{chain_id="limonata_10777-1",instance="54.39.128.229:27660",job="limonata_validator"} 39

# HELP cometbft_mempool_tx_size_bytes Histogram of transaction sizes in bytes
# TYPE cometbft_mempool_tx_size_bytes histogram
cometbft_mempool_tx_size_bytes_bucket{chain_id="limonata_10777-1",instance="54.39.128.229:27660",job="limonata_validator",le="729"} 40
cometbft_mempool_tx_size_bytes_sum{chain_id="limonata_10777-1",instance="54.39.128.229:27660",job="limonata_validator"} 12155
cometbft_mempool_tx_size_bytes_count{chain_id="limonata_10777-1",instance="54.39.128.229:27660",job="limonata_validator"} 40

# HELP cometbft_abci_connection_method_timing_seconds Timing for each ABCI method
# TYPE cometbft_abci_connection_method_timing_seconds histogram
cometbft_abci_connection_method_timing_seconds_bucket{chain_id="limonata_10777-1",method="commit",type="sync",le="0.1"} 480
cometbft_abci_connection_method_timing_seconds_bucket{chain_id="limonata_10777-1",method="commit",type="sync",le="+Inf"} 500
cometbft_abci_connection_method_timing_seconds_sum{chain_id="limonata_10777-1",method="commit",type="sync"} 47.6
cometbft_abci_connection_method_timing_seconds_count{chain_id="limonata_10777-1",method="commit",type="sync"} 500
```

---

*Document maintained by [Cumulo](https://cumulo.pro), Limonata Testnet Validator*
*Chain: `limonata_10777-1` | Validator: `Cumulo` | Updated: July 2026*
