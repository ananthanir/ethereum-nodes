# qbft-eth-nw

A local 4-node private Ethereum network running [Hyperledger Besu](https://besu.hyperledger.org/) with [QBFT](https://besu.hyperledger.org/private-networks/how-to/configure/consensus/qbft) consensus, via Docker Compose. Useful for local dev/testing of contracts or tooling against a fast, disposable chain (chain ID `1337`).

## Requirements

- Docker and Docker Compose v2

## Layout

```
docker-compose.yaml       # the 4 besu nodes
network/genesis.json      # shared genesis / QBFT config used by all nodes
network/Node-1..4/data     # per-node identity (validator key) + chain data
network/networkFiles/      # reference copy of the config used to generate the above
network/cmd.txt            # equivalent bare `besu` commands, if you'd rather run without Docker
```

`node1` (`192.168.1.100:30303`) acts as the bootnode; nodes 2–4 connect to it and discover each other via QBFT/devp2p from there.

## Running

```bash
docker compose up -d
docker compose logs -f
```

Give it a few seconds — QBFT block production starts once all 4 validators are up and can see each other. Block time is 2 seconds (`blockperiodseconds` in `network/genesis.json`).

Check block production:

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://localhost:8545
```

Check the validator set:

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"qbft_getValidatorsByBlockNumber","params":["latest"],"id":1}' \
  http://localhost:8545
```

## Endpoints

| Node  | JSON-RPC (HTTP) | JSON-RPC (WS) | P2P   |
|-------|-----------------|----------------|-------|
| node1 | 8545            | 9545           | 30303 |
| node2 | 8546            | 9546           | 30304 |
| node3 | 8547            | 9547           | 30305 |
| node4 | 8548            | 9548           | 30306 |

All nodes expose `ETH,NET,QBFT,ADMIN,DEBUG,MINER,TXPOOL,WEB3,TRACE` RPC namespaces and allow CORS/any host — this is a **local dev network only**, never expose these ports publicly.

## Funded test accounts

Three accounts are pre-funded in `network/genesis.json` for local testing. Their private keys are committed in plaintext in the genesis file — this is normal for disposable local dev chains but **must never be reused on any real network**.

## Resetting the chain

The per-node `data/` folders hold the actual chain state, separate from the validator identity keys (`key`/`key.pub`, which must stay put — they define the validator set in `genesis.json`'s `extraData`). To wipe the chain and start over from block 0:

```bash
docker compose down
rm -rf network/Node-*/data/database network/Node-*/data/caches
docker compose up -d
```

(`network/.gitignore` already excludes these paths from version control.)

## Running without Docker

If you have `besu` installed locally, `network/cmd.txt` has the equivalent standalone commands for each node (Windows paths, run from inside `network/`).

## Regenerating the network from scratch

`network/networkFiles/` contains the Besu [QBFT config generator](https://besu.hyperledger.org/private-networks/tutorials/qbft) input (`qbftConfigFile.json`) used to originally produce the genesis file and validator keys. Re-run it only if you want a fresh validator set — doing so invalidates the existing `Node-*/data` identities and any existing chain data.
