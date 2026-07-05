# Limonata: Reference Commands

Quick reference for day-to-day operation of the Limonata node/validator (`limonata_10777-1`) by Cumulo. Assumes binary `evmd`, home at `/home/<server_user>/.limonata`, RPC on local port `26677`.

## Sync status

```bash
curl -s localhost:26677/status | jq '.result.sync_info'
```

Compare against an external RPC to rule out a false positive (node stuck at a low block with `catching_up: false`):

```bash
curl -s https://cosmos-rpc.limonata.xyz/status | jq -r '.result.sync_info.latest_block_height'
```

## Peers

```bash
curl -s localhost:26677/net_info | jq '.result.n_peers'
```

## Validator status

```bash
evmd query staking validator <cosmosvaloper1...> --node tcp://localhost:26677
```

## Account balance

```bash
evmd query bank balances <cosmos1...> --node tcp://localhost:26677
```

## Delegate

```bash
evmd tx staking delegate <valoper> <amount>aLIMO \
  --from=<key> --keyring-backend=test --home=<home> \
  --chain-id=limonata_10777-1 --gas=auto --gas-adjustment=1.4 \
  --fees=10000000aLIMO --node=tcp://localhost:26677 -y
```

## Unjail

```bash
evmd tx slashing unjail --from=<key> --keyring-backend=test \
  --home=<home> --chain-id=limonata_10777-1 --gas=auto --gas-adjustment=1.4 \
  --fees=10000000aLIMO --node=tcp://localhost:26677 -y
```

## Query a transaction

```bash
evmd query tx <txhash> --node tcp://localhost:26677
```

## Reset chain data (keeps keys)

```bash
evmd comet unsafe-reset-all --home <home> --keep-addr-book
```

> Does not touch `priv_validator_key.json` or `node_key.json`, only clears blockchain/state data.

## Check ports in use (relevant range)

```bash
sudo ss -tlnp | grep -E ':(26[0-9]{3}|9[0-9]{3}|8[0-9]{3}|1[0-9]{3})\b'
```

## Live logs

```bash
journalctl -u <service> -f
```

## systemd management

Unit file: `/etc/systemd/system/limonatad.service`

```bash
sudo systemctl daemon-reload
sudo systemctl enable limonatad
sudo systemctl restart limonatad && sudo journalctl -u limonatad -fo cat
sudo systemctl stop limonatad
```

## Validator

```bash
# Node status
evmd status --node tcp://localhost:26677 2>&1 | jq

# Consensus pubkey
evmd comet show-validator

# Binary version
evmd version
```

## Wallet

```bash
# Add a wallet
evmd keys add cumwallet

# Check balance
evmd query bank balances cumwallet
```
