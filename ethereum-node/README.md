# ethereum-node

A minimal Ethereum full node: [Geth](https://geth.ethereum.org/) (execution client) + [Lighthouse](https://lighthouse.sigp.io/) (consensus client), wired together with Docker Compose and the post-Merge Engine API.

## Requirements

- Docker and Docker Compose v2 (`docker compose version`)
- Disk space: mainnet requires **2+ TB** of fast SSD/NVMe storage (snap-synced, pruned). Growing steadily over time.
- A stable, always-on connection if you want to keep up with head.

## How it fits together

- `geth` runs the execution client, exposing JSON-RPC (`8545`), WebSocket (`8546`), and the authenticated Engine API (`8551`) used by the consensus client.
- `lighthouse` runs the beacon node, syncing consensus data and driving `geth` via the Engine API. It uses [checkpoint sync](https://lighthouse-book.sigmaprime.io/advanced_checkpoint_sync.html) against Sigma Prime's public endpoint so you don't have to sync consensus history from genesis.
- Both clients share a JWT secret (`geth/jwt.hex`) to authenticate Engine API calls between them, per the [post-Merge client spec](https://github.com/ethereum/execution-apis/blob/main/src/engine/authentication.md).

## First-time setup

1. Generate a JWT secret (skip if `geth/jwt.hex` already exists):

   ```bash
   openssl rand -hex 32 > geth/jwt.hex
   ```

2. Start the stack:

   ```bash
   docker compose up -d
   ```

3. Watch the logs while it syncs:

   ```bash
   docker compose logs -f
   ```

   Geth will snap-sync execution data; Lighthouse will checkpoint-sync consensus data. Initial sync can take anywhere from a few hours to a day or more depending on bandwidth and disk speed.

## Endpoints

| Service              | Port  | Purpose                          |
|-----------------------|-------|-----------------------------------|
| Geth JSON-RPC (HTTP)  | 8545  | `eth`, `net`, `web3`, `txpool`     |
| Geth JSON-RPC (WS)    | 8546  | Same namespaces over WebSocket     |
| Geth P2P              | 30303 | Execution-layer peer discovery     |
| Geth Engine API       | 8551  | Internal use only (geth ↔ lighthouse) |
| Lighthouse HTTP API   | 5052  | Beacon node REST API               |
| Lighthouse P2P        | 9000  | Consensus-layer peer discovery     |

Sanity check once it's up:

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://localhost:8545

curl -s http://localhost:5052/eth/v1/node/syncing
```

## Switching networks

By default this runs **mainnet**. To run a testnet instead (e.g. Sepolia or Holesky):

1. In `docker-compose.yml`, add the matching flag to the `execution` service's `command` (e.g. `"--sepolia"`).
2. Change `--network=` in the `consensus` service's `command` to match (e.g. `--network=sepolia`).
3. Update `--checkpoint-sync-url` to a checkpoint sync provider for that network — see the [checkpoint sync provider list](https://eth-clients.github.io/checkpoint-sync-endpoints/).

## Data and secrets

- Chain data lives in `geth/data` and `lighthouse/data` (bind-mounted, git-ignored).
- `geth/jwt.hex` and `lighthouse/jwt.hex` are local secrets used only for local Engine API auth between the two containers on your machine — they are git-ignored and never need to leave your host. If you ever suspect it leaked, just regenerate it (step 1 above) and restart both containers.

## Stopping / resetting

```bash
docker compose down          # stop containers, keep chain data
docker compose down -v       # stop and also remove any named volumes (bind-mounted data dirs are untouched — delete geth/data and lighthouse/data manually to fully reset)
```
