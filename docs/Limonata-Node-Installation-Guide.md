# Limonata: Node Installation Guide

Installation and bootstrap of a Limonata full node (`limonata_10777-1`, EVM chain-id `10777`) for operation by Cumulo.

## 1. Network info

| | |
|---|---|
| **Chain** | Limonata (EVM L1 on Cosmos SDK) |
| **Chain-ID (cosmos)** | `limonata_10777-1` |
| **Chain-ID (EVM)** | `10777` |
| **Denom** | `aLIMO` |
| **Binary** | `evmd` (framework's internal name, not rebranded) |
| **Default home** | `~/.evmd` |
| **Bech32 prefix** | `cosmos1...` (not rebranded to `limonata1...`) |

## 2. Official sources

- Repo: https://github.com/Limonata-Blockchain/limonata
- Validator guide: https://limonata.xyz/VALIDATOR.md
- Genesis: https://limonata.xyz/genesis.json
- Faucet: `POST https://limonata.xyz/api/squeeze/fund`
- State-sync RPC: https://cosmos-rpc.limonata.xyz
- Seed: `4b154368aab24cb5b31c927efd50c73d0f4f9799@142.127.103.79:26656`

## 3. Requirements

- Go 1.25.9+
- Free ports (see §5): check before assigning:
  ```
  sudo ss -tlnp | grep -E ':(26[0-9]{3}|9[0-9]{3}|8[0-9]{3}|1[0-9]{3})\b'
  ```

## 4. Binary

There are two relevant releases. **v0.1.0 is the one used to sync from genesis**: v0.2.0 cannot replay blocks prior to the upgrade (block `766558`, introduces the `x/vpcap` module).

### 4.1 Build v0.1.0 (required for sync)

```bash
git clone https://github.com/Limonata-Blockchain/limonata.git limonata-v1
cd limonata-v1
git checkout limonata-testnet-v0.1.0
make install
# The binary is named 'evmd', not 'limonatad': installs to $GOPATH/bin
```

### 4.2 Prebuilt v0.2.0 binary (staged for the future upgrade, do not use yet)

```bash
wget https://github.com/Limonata-Blockchain/limonata/releases/download/limonata-testnet-v0.2.0/limonatad-linux-amd64.tar.gz
wget https://github.com/Limonata-Blockchain/limonata/releases/download/limonata-testnet-v0.2.0/SHA256SUMS.txt
tar xzf limonatad-linux-amd64.tar.gz
sha256sum -c SHA256SUMS.txt
sudo install limonatad /usr/local/bin/
```

> ⚠️ Extract (`tar`) **before** verifying (`sha256sum -c`): otherwise the binary listed in `SHA256SUMS.txt` doesn't exist on disk yet and verification fails.

## 5. Ports

Multi-chain server: ports reassigned to avoid collisions.

| Service | Port | Config |
|---|---|---|
| P2P | `26676` | `config.toml` → `[p2p] laddr` |
| RPC | `26677` | `config.toml` → `[rpc] laddr` |
| gRPC | `9096` | `app.toml` → `[grpc] address` |
| API (REST) | `1327` | `app.toml` → `[api] address` |
| JSON-RPC | `8545` | `app.toml` → `[json-rpc] address` |
| WS JSON-RPC | `8546` | `app.toml` → `[json-rpc] ws-address` |

## 6. Initialization

```bash
evmd init <moniker> --chain-id limonata_10777-1 --home /home/dedicaovh/.limonata
wget https://limonata.xyz/genesis.json -O /home/dedicaovh/.limonata/config/genesis.json
```

Required changes in `config.toml`:

```toml
seeds = "4b154368aab24cb5b31c927efd50c73d0f4f9799@142.127.103.79:26656"

[mempool]
type = "app"  # required on this build
```

## 7. systemd

```ini
[Unit]
Description=Limonata Testnet Node (Cumulo)
After=network-online.target
Wants=network-online.target

[Service]
User=dedicaovh
WorkingDirectory=/home/dedicaovh/.limonata
ExecStart=/home/dedicaovh/go/bin/evmd start \
  --home /home/dedicaovh/.limonata \
  --chain-id limonata_10777-1 \
  --evm.evm-chain-id 10777 \
  --minimum-gas-prices 0aLIMO
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

> `WorkingDirectory` is mandatory: without it, systemd starts with cwd at `/` and fails with `mkdir data: permission denied` when the binary uses relative paths. Never use inline comments in systemd directives (e.g. `CPUQuota=50% # note` silently breaks the limit).

```bash
sudo systemctl daemon-reload
sudo systemctl enable limonatad
sudo systemctl start limonatad
journalctl -u limonatad -f
```

## 8. Sync

**Do not sync from genesis or state-sync with the v0.2.0 binary**: it fails deterministically (`wrong Block.Header.AppHash` at block 2, or `multistore restore: version of store vpcap mismatch` on state-sync) because the `vpcap` store doesn't exist in the chain's actual history. Use **v0.1.0** for the whole sync process.

### State-sync (recommended)

```toml
[statesync]
enable = true
rpc_servers = "https://cosmos-rpc.limonata.xyz:443,https://cosmos-rpc.limonata.xyz:443"
trust_height = <recent height>
trust_hash = "<header hash at that height>"
trust_period = "168h0m0s"
```

To get the correct `trust_hash` (the `Discovered new snapshot` hash in the logs is **not** it: that's the snapshot hash, not the header hash):

```bash
curl -s "https://cosmos-rpc.limonata.xyz/block?height=<H>" | jq -r '.result.block_id.hash'
```

```bash
evmd comet unsafe-reset-all --home /home/dedicaovh/.limonata --keep-addr-book
sudo systemctl restart limonatad
```

> `unsafe-reset-all` doesn't touch `priv_validator_key.json` or `node_key.json`, it only clears blockchain/state data.

### Verification

```bash
curl -s localhost:26677/status | jq '.result.sync_info'
```

Always compare against an external RPC: a node stuck at a low block with `catching_up: false` is a false positive:

```bash
curl -s https://cosmos-rpc.limonata.xyz/status | jq -r '.result.sync_info.latest_block_height'
```

## 9. Next step

Once the node is synced, continue with key and validator creation → see [`validator/`](../validator).
