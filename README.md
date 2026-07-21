# TSN — Public Test Network

**Trust Stack Network** is a post-quantum, privacy-preserving proof-of-work
blockchain: ML-DSA-65 signatures (FIPS 204), shielded transactions with
zero-knowledge proofs, and verified state snapshots.

This repository distributes **release binaries only**. No source code.

---

## Status

The public test network is **live and open**. Anyone can run a node or mine.
Coins on this network have **no value** and the chain may be reset without
notice.

| | |
|---|---|
| Network | `tsn-devnet-v2-gen4` |
| Chain ID | `28ee68e414994c39` |
| Genesis | `15285cf55964230efe13450bc520bd46bc023469e4d207f91d22daaa56fbef78` |
| Current release | `v3.0.0-rc.10` |
| Block time | ~10 seconds |

---

## Download

| Platform | File |
|---|---|
| Linux x86_64 | `tsn-linux-x86_64` |

Windows and macOS builds are in preparation.

**Requirements:** Linux x86_64, glibc 2.34 or newer (Ubuntu 22.04+, Debian 12+,
Fedora 36+). Tested on Ubuntu 24.04 and 26.04. About 1 GB of free disk space.

### Verify your download

```bash
sha256sum tsn-linux-x86_64
```

Expected:

```
e2481f74d030528b0ccff502d18f4876c4d5c26f0da7905f58fc051e7e102c89
```

Do not run the binary if the checksum does not match.

```bash
chmod +x tsn-linux-x86_64
./tsn-linux-x86_64 --version
```

---

## ⚠️ Use the DevnetV2 commands

The binary still ships legacy commands that target a **retired network**. They
will connect to nothing and fail with `0/5 peers responded` — this does not mean
the network is empty, it means you are on the wrong one.

| Use these | Not these |
|---|---|
| `service-node`, `miner-v2` | `node`, `relay`, `miner`, `mine` |

Anything marked `(DevnetV2)` in `--help` is correct.

---

## Run a node

A node follows the chain, stores blocks and relays them to other peers. It does
not mine and needs no wallet.

```bash
./tsn-linux-x86_64 service-node \
  --port 9437 \
  --bind 127.0.0.1 \
  --data-dir ./data \
  --peer /dns4/seed1.tsnchain.com/tcp/9433 \
  --peer /dns4/seed2.tsnchain.com/tcp/9433 \
  --peer /dns4/seed3.tsnchain.com/tcp/9433 \
  --peer /dns4/seed4.tsnchain.com/tcp/9433 \
  --peer /dns4/nexus.tsnchain.com/tcp/9433
```

On first start the node bootstraps from the latest signed snapshot instead of
replaying the chain, so it is caught up within a few minutes. The snapshot is
verified against the network's signing keys and its published checksum; if
anything fails it falls back to a normal sync.

Check its status at any time:

```bash
curl -s http://127.0.0.1:9437/health
```

You are fully synced when `tip_height` matches the network (compare with
https://explorer.tsnchain.com). Note that `p2p_peers_accepted` is a cumulative
counter of accepted handshakes, not the number of live connections.

### Running behind NAT

No port forwarding is required. The node opens outbound connections to the
seeds; it does not need to be reachable from the internet.

If you *want* your node to be reachable by others, bind a public address and
add `--accept-public-launch`.

---

## Mine

### 1. Create a wallet

```bash
./tsn-linux-x86_64 new-wallet -o ./wallet.json
```

This prints a **24-word recovery phrase**. Write it down and store it offline —
it is shown only once, and it is the only way to recover your coins.

The command then prints an `Address:` value. **That value is your miner public
key hash** — you will pass it as `--miner-pk-hash` below. You can also read it
back from the `pk_hash` field of `wallet.json`.

Keep `wallet.json` safe and private (it is created with `0600` permissions).

### 2. Start mining

Replace `<YOUR_PK_HASH>` with the value from step 1.

```bash
./tsn-linux-x86_64 miner-v2 \
  --port 18546 \
  --bind 127.0.0.1 \
  --p2p-bind 127.0.0.1 \
  --p2p-port 19434 \
  --data-dir ./miner-data \
  --miner-pk-hash <YOUR_PK_HASH> \
  --peer /dns4/seed1.tsnchain.com/tcp/9433 \
  --peer /dns4/seed2.tsnchain.com/tcp/9433 \
  --peer /dns4/seed3.tsnchain.com/tcp/9433 \
  --peer /dns4/seed4.tsnchain.com/tcp/9433 \
  --peer /dns4/nexus.tsnchain.com/tcp/9433
```

The miner first syncs the chain, then waits until it is connected to the gossip
mesh before producing its first block. This is intentional: it prevents mining
blocks that nobody would receive.

Watch progress with:

```bash
curl -s http://127.0.0.1:18546/health
```

- `pow_hashes_tried` rising → you are mining.
- `blocks_mined` rising → you are winning blocks.
- `miner_wait_blocks_graft` in the logs → still waiting for the gossip mesh,
  this is normal during the first minute.

## Troubleshooting

**`observe_peers: only 0/5 peers responded` / `Timed out waiting for sync`**

You are running a legacy command. Those target the retired network on port 9333
and will never find peers, however many nodes are online. Use `service-node` or
`miner-v2` instead — see the warning at the top of this page.

**`miner_wait_blocks_graft ... grafted=0`**

The miner is waiting to be grafted onto the `blocks` gossip topic before
producing, so it never mines a block into the void. It normally clears within a
minute. If it persists after a restart, check that outbound TCP to port 9433 on
the seed addresses is not blocked.

**Sync appears stuck at height 0 for several minutes**

Normal: the node opens and validates its database before serving HTTP. Full sync
from genesis takes several minutes before the height starts moving.

Mining is CPU-based. Use `-t <N>` to set the number of threads.

---

## Claim your rewards

Block rewards are credited to your `pk_hash` and must be claimed **per block**.
You need the height of a block you mined — `blocks_mined` in `/health` tells you
how many you won, and the miner logs the height of each one.

```bash
./tsn-linux-x86_64 claim-coinbase --block <HEIGHT> --wallet ./wallet.json
```

Check your balance (point it at your own miner, or at the public indexer):

```bash
./tsn-linux-x86_64 balance --wallet ./wallet.json --node http://127.0.0.1:18546
```

`--block` is required — a bare `claim-coinbase --wallet ...` will fail.

---

## Security

- Never share `wallet.json` or your 24-word phrase. Anyone holding either
  controls your coins.
- Keep the HTTP port bound to `127.0.0.1`. It is an administration interface.
- Always verify the SHA-256 checksum before running a new release.

Report vulnerabilities privately rather than opening a public issue.

---

## Links

- Website: https://tsnchain.com
- Explorer: https://explorer.tsnchain.com
- Snapshots: https://github.com/trusts-stack-network/tsn-snapshots

---

## License

The binaries are distributed for testing on the public test network. Source code
is not published in this repository.
