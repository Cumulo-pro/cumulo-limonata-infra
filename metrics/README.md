# Limonata Testnet: Monitoring & Metrics

This document describes Cumulo's monitoring stack for the Limonata Testnet validator infrastructure, based on **CometBFT native Prometheus metrics + node_exporter -> Prometheus -> Grafana**.

---

## Architecture

```
+-----------------------------------------------------------------+
|                       VALIDATOR NODE                             |
|  ns562220 (OVH)                                                  |
|                                                                   |
|  limonatad (evmd binary) --Prometheus--> :27660 /metrics         |
|  node_exporter -----------------------> :9100  /metrics          |
+-------------------------------+-----------------------------------+
                                |
+-----------------------------------------------------------------+
|                          RPC NODE                                |
|  Velia2                                                          |
|                                                                   |
|  limonatad --Prometheus--> :26661 /metrics   (planned)           |
|  node_exporter ----------> :9100  /metrics                       |
+-------------------------------+-----------------------------------+
                                |
                                v
                    +--------------------------+
                    |    Central Prometheus    |
                    |      (port 9099)         |
                    +------------+--------------+
                                 |
                                 v
                          +--------------+
                          |    Grafana   |
                          +--------------+
```

Unlike Gnoland (TM2), Limonata's binary (`evmd`, based on CometBFT) exposes Prometheus metrics **natively** - no OpenTelemetry Collector is required in this stack. Prometheus scrapes the node's `/metrics` endpoint directly.

---

## Components

### 1. CometBFT native metrics

Prometheus is enabled directly in `config.toml`:

```toml
[instrumentation]
prometheus = true
prometheus_listen_addr = ":27660"
max_open_connections = 3
namespace = "tendermint"
```

> Despite `namespace = "tendermint"`, consensus/mempool/p2p/abci metrics are exposed under the `cometbft_` prefix in this build - the `namespace` field does not override it. Always verify exact metric names against the live endpoint rather than assuming the config value:
> ```
> curl -s localhost:27660/metrics | grep "^# HELP"
> ```

Restart after any config change:

```
sudo systemctl daemon-reload
sudo systemctl restart limonatad
curl -s localhost:27660/metrics | head
```

### 2. node_exporter (system metrics)

Standard `node_exporter` running on the same host, already shared across all chains hosted on the server. No Limonata-specific setup required beyond confirming the target is scraped.

**Port:** `9100`

### 3. Central Prometheus

**Endpoint:** `http://<PROMETHEUS_IP>:9099`

Add the following scrape job to `prometheus.yml`:

```yaml
scrape_configs:

  - job_name: limonata_validator
    static_configs:
      - targets: ['<VALIDATOR_NODE_IP>:27660']
```

Reload after editing:

```
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl reload prometheus
```

Confirm the target is up:

```
curl -s http://localhost:9099/api/v1/targets | grep -A5 limonata
```

### 4. Grafana Dashboard

**Dashboard file:** [`limonata-grafana-dashboard.json`](https://github.com/Cumulo-pro/cumulo-limonata-infra/blob/main/metrics/limonata_grafana_dashboard.json)

The dashboard covers:

- **Node overview** - sync status, block height, peers, voting power, missed blocks
- **Block metrics** - block height over time, sync speed, block interval, consensus rounds, transactions
- **Validator & signing** - voting power share, recent missed blocks (rate-based, not cumulative), signing lag
- **Mempool** - reaped txs, tx throughput, tx size, duplicate tx receptions, batch size
- **P2P network** - connected peers, receive/send bandwidth
- **ABCI performance** - per-method latency (commit, finalize_block, prepare_proposal, etc.)
- **System resources** - CPU, memory, disk, network traffic, load average (via node_exporter)

A `job` template variable (multi-select) lets you switch between or compare multiple Limonata nodes without editing queries.

---

## Public Endpoints

| Resource               | URL                                        |
| ----------------------- | ------------------------------------------- |
| RPC (testnet)            | `https://rpc-testnet.limonata.cumulo.me`    |
| API (testnet)            | `https://api-testnet.limonata.cumulo.me`    |
| EVM JSON-RPC (testnet)   | `https://evm-testnet.limonata.cumulo.me`    |
| gRPC (testnet)           | `https://grpc-testnet.limonata.cumulo.me`   |
| Prometheus metrics (validator) | `http://<VALIDATOR_NODE_IP>:27660/metrics` |
| node_exporter (validator)      | `http://<VALIDATOR_NODE_IP>:9100/metrics`  |

---

## Key Metrics

Limonata (`evmd`, CometBFT-based) exposes metrics under the `cometbft_` prefix. Key metrics available:

| Metric                                          | Description                                       |
| ------------------------------------------------ | -------------------------------------------------- |
| `cometbft_consensus_height`                       | Current block height                                |
| `cometbft_consensus_validators`                   | Number of validators in the active set              |
| `cometbft_consensus_validators_power`              | Total voting power of the network                   |
| `cometbft_consensus_validator_power`               | This validator's voting power                       |
| `cometbft_consensus_validator_missed_blocks`       | Cumulative missed blocks since process start (counter - use `increase()`, not the raw value, to check current signing health) |
| `cometbft_consensus_validator_last_signed_height`  | Last height signed by this validator                |
| `cometbft_consensus_byzantine_validators`          | Byzantine validators detected in the last commit     |
| `cometbft_consensus_rounds`                        | Number of consensus rounds per block                 |
| `cometbft_consensus_block_interval_seconds`        | Histogram of time between blocks                     |
| `cometbft_consensus_total_txs`                     | Total transactions processed (cumulative)            |
| `cometbft_consensus_num_txs`                       | Transactions in the latest block                     |
| `cometbft_mempool_reaped_txs`                      | Total transactions reaped from the mempool           |
| `cometbft_mempool_already_received_txs`            | Duplicate transaction receptions (gossip efficiency)  |
| `cometbft_mempool_tx_size_bytes`                   | Histogram of transaction sizes                        |
| `cometbft_mempool_batch_size`                      | Histogram of inbound/outbound tx batch sizes          |
| `cometbft_p2p_peers`                                | Number of connected peers                             |
| `cometbft_p2p_peer_receive_bytes_total`             | P2P bytes received                                    |
| `cometbft_p2p_peer_send_bytes_total`                | P2P bytes sent                                        |
| `cometbft_abci_connection_method_timing_seconds`    | ABCI method latency histogram (commit, finalize_block, etc.) |

> This build does not expose a live mempool size gauge (`mempool_size` / `mempool_size_bytes`). Use `cometbft_mempool_reaped_txs` and its rate as the closest available proxy for mempool activity.

> All metrics carry the `instance` and `job` labels, plus `chain_id="limonata_10777-1"`.

Verify metrics are being received:

```
curl -s http://<VALIDATOR_NODE_IP>:27660/metrics | grep -v "^#" | head -20
```

---

## Troubleshooting

**Missed blocks counter looks wrong (high even though the explorer shows 100% uptime)**

- `cometbft_consensus_validator_missed_blocks` is cumulative since the process started. During historical replay/state-sync (before the validator was bonded), every pre-bonding block is counted as "missed" even though the validator was never expected to sign it.
- Always monitor `increase(cometbft_consensus_validator_missed_blocks[10m])` instead of the raw counter. It should sit at 0 once the node is fully synced.
- Cross-check against the block explorer's signing/uptime page as the source of truth for historical signing performance.

**Wrong config.toml edited, metrics still not appearing after restart**

- Confirm the actual home directory in use by the running process before editing any config:
  ```
  cat /proc/<PID>/cmdline | tr '\0' ' '
  ```
- Limonata's home is `~/.limonata`, not the `~/.evmd` default implied by the binary name.

**Port conflicts on shared multi-chain servers**

- `26660` is a common default across chains; check occupied ports before assigning:
  ```
  sudo ss -tulpn | grep -E ':2[6-9][0-9]{3}'
  ```
- Limonata validator uses `27660` for Prometheus on `ns562220` to avoid collisions with other colocated chains.

**Target not appearing in Prometheus**

- Verify firewall allows the metrics port: `sudo ufw allow 27660/tcp`
- Test connectivity from the Prometheus server: `curl -s http://<VALIDATOR_NODE_IP>:27660/metrics | head -5`
- Confirm the job name and target match what is configured in `prometheus.yml`:
  ```
  curl -s http://localhost:9099/api/v1/targets | grep -A5 limonata
  ```

---

*Maintained by [Cumulo](https://cumulo.pro) - validator infrastructure for Limonata Testnet*
