---
sidebar_position: 2
title: Join the network
---

# Running an Earth node

This page contains the full procedure to sync a node on `earth-1`.

If you cannot sync a node with this page alone, that is a defect in this page.
Open an issue.

> **The network launched at 2026-08-28T06:30:00Z**, from the genesis in the
> `v0.5.0` release. It has one validator so far, so a node joining today syncs
> from that one peer and adds the second.

---

## 1. Get the binary

Download from the [latest release](https://github.com/zenopie/earth-network-chain/releases/latest):

```bash
VERSION=v0.5.0          # the launch tag
ARCH=amd64              # or arm64

curl -LO https://github.com/zenopie/earth-network-chain/releases/download/$VERSION/earthd_${VERSION}_linux_${ARCH}.tar.gz
curl -LO https://github.com/zenopie/earth-network-chain/releases/download/$VERSION/checksums.txt

sha256sum -c checksums.txt --ignore-missing     # the result must be OK
tar xzf earthd_${VERSION}_linux_${ARCH}.tar.gz

sudo install -m755 earthd_${VERSION}_linux_${ARCH}/bin/earthd /usr/local/bin/
sudo install -m644 earthd_${VERSION}_linux_${ARCH}/lib/*     /usr/local/lib/
```

Run both `install` commands. `earthd` links `libwasmvm`, the CosmWasm engine, as
a shared library. The two components are version-locked. Do not use the `earthd`
of one release with the `libwasmvm` of another release. The tarball contains the
correct copy in `lib/`, with the C++ runtime that the proof verifier requires.

The binary looks for these libraries at `../lib`, relative to its own location.
Installation to `/usr/local/bin` and `/usr/local/lib` therefore needs no
`ldconfig` and no `LD_LIBRARY_PATH`. If you install to a different location, keep
the same relative layout.

Check the installation:

```bash
earthd version --long
```

Compare the `version` and `commit` values with the values that other operators
run. This is the important check during an upgrade.

**To build instead of download**, you need cgo and the proof verifier:

```bash
sudo apt-get install -y clang python3 binutils libc++-dev libc++abi-dev
cd third_party/barretenberg-go && ./scripts/build-wrapper.sh --platform linux_amd64
cd ../.. && make install
```

A container image is also available at `ghcr.io/zenopie/earth-network-chain`,
pinned by digest. See [docker/README.md](https://github.com/zenopie/earth-network-chain/blob/master/docker/README.md).

---

## 2. Initialise

```bash
earthd init "<your-moniker>" --chain-id earth-1
```

---

## 3. Install and verify the genesis file

This step determines whether you join `earth-1` or start a separate chain. A
genesis file that differs by one byte produces a different app hash. That node
never reaches agreement with the network.

```bash
curl -L -o ~/.earth/config/genesis.json \
  https://github.com/zenopie/earth-network-chain/releases/download/$VERSION/genesis.json

sha256sum ~/.earth/config/genesis.json
```

The output must be:

```
3701aa69c304f45bbede5bb9eef3b7770d57dd7c03f39caa8c1d7b8a1ea4f792  genesis.json
```

A genesis that hashes to anything else is a different chain, whatever its
`chain_id` says.

If the value differs, stop. Do not continue.

```bash
earthd genesis validate-genesis
```

---

## 4. Configure

**Seeds and reachability**, in `~/.earth/config/config.toml`:

```toml
# persistent_peers, not seeds. A seed is a crawler that hands out addresses and
# disconnects; this is the network's one node, and you want to hold a connection
# to it. There is no seed node yet, and seed.erth.network does not resolve.
persistent_peers = "6ad7347a1e15cc3a247347e152fee9fdd7ed2440@provider.akash-palmito.org:32688"
# The address that other nodes use to reach this node. Set it if the node is
# behind NAT, in a container, or at a provider that maps ports. If it is unset,
# CometBFT advertises the address that it observes on itself and gives that
# address to each peer. Other nodes then cannot dial this node.
external_address = "your.host.or.ip:26656"
```

Port **26656** must accept inbound connections. A node without inbound
connectivity can still sync, because it dials out. But no peer can dial it. It
therefore adds no connectivity to the network and cannot serve state sync.

The node id above is fixed: it is derived from a node key the deployment injects,
so it survives a redeploy. The host and port are the Akash provider's, and the
port is assigned by the provider — if it ever changes, this page is wrong and the
value in the node's own `/status` under `listen_addr` is right.

For the Docker image, use the `SEEDS`, `PERSISTENT_PEERS`, and `EXTERNAL_ADDRESS`
environment variables. The entrypoint writes them into `config.toml` at each
start, so a restart applies a change.

**Minimum gas price**, in `~/.earth/config/app.toml`. This value is **required.
The node does not start without it.** The error message does not name the file:

```
set min gas price in app.toml or flag or env variable
```

The node does not relay a transaction below this value. This is a per-node
setting, not a chain rule. There is no fee module, so the effective floor of the
network is the value that most validators select:

```toml
minimum-gas-prices = "0.005uerth"
```

**Pruning**, in `app.toml`. Select by the role of the node:

| Role | Setting |
| --- | --- |
| Validator | `pruning = "default"` |
| Public RPC | `pruning = "custom"`, `pruning-keep-recent = "362880"`, `pruning-interval = "100"` |
| Archive | `pruning = "nothing"` — the disk requirement has no limit |

**Snapshots** are enabled by default. Keep them enabled:

```toml
snapshot-interval = 1000      # approximately 80 minutes at 5-second blocks
snapshot-keep-recent = 5
```

Snapshots permit *other* operators to state-sync from this node. A value of `0`
disables them. If all operators disable them, a new node must replay the
complete chain.

If you change `pruning`, confirm that the node still keeps the states that a
snapshot requires. `pruning = "default"` keeps more states than the snapshot
interval requires, so the two settings do not conflict.

Do **not** enable `enabled-unsafe-cors` on a validator. It permits any website to
read the node and to broadcast through it. To serve a browser application, run a
separate read-only node.

---

## 4b. State sync (optional, and much faster)

State sync fetches state at a recent height from a peer. It does not replay
each block.

**This chain gains more from state sync than most chains.** A replay re-executes
the transactions of each block, and each passport registration verifies a
zero-knowledge proof. A chain that only transfers tokens replays quickly. This
chain re-runs one proof for each registration, so replay cost increases with
adoption.

Get a trust height and hash below the current tip:

```bash
RPC=https://rpc.erth.network
LATEST=$(curl -s $RPC/block | jq -r .result.block.header.height)
TRUST_HEIGHT=$(( LATEST - 2000 ))
TRUST_HASH=$(curl -s "$RPC/block?height=$TRUST_HEIGHT" | jq -r .result.block_id.hash)
echo "$TRUST_HEIGHT  $TRUST_HASH"
```

Enter the values in `~/.earth/config/config.toml`, under `[statesync]`:

```toml
enable = true
rpc_servers = "https://rpc.erth.network:443,https://rpc.erth.network:443"
trust_height = <TRUST_HEIGHT>
trust_hash = "<TRUST_HASH>"
trust_period = "168h0m0s"
```

`rpc_servers` requires **two entries or more**. The same address two times is
accepted. Two independent addresses are better.

Then start with an empty data directory:

```bash
earthd tendermint unsafe-reset-all --home ~/.earth --keep-addr-book
earthd start
```

The log shows `Discovering snapshots`, then `Fetching snapshot chunks`. If it
stays at `Discovering snapshots`, no peer offers a snapshot. Confirm that
`snapshot-interval` is not zero on the source node.

---

## 5. Start

```bash
earthd start
```

Monitor progress:

```bash
curl -s localhost:26657/status | jq .result.sync_info
```

`catching_up: false` indicates that the node is synced.

---

## Hardware

| | Validator | Public RPC | Archive |
| --- | --- | --- | --- |
| CPU | 4 cores | 4 cores | 8 cores |
| RAM | 16 GB | 16 GB | 32 GB |
| Disk | 500 GB SSD | 1 TB SSD | 2 TB or more, SSD |

Use an SSD. Do not use a rotating disk. The node calls fsync at each block.

**One requirement is specific to this chain.** Each passport registration
verifies a zero-knowledge proof on-chain. This uses significant CPU time and the
node cannot omit it. Allocate more CPU capacity than a chain of this size usually
requires.

---

## How to become a validator

Sync the node first. A node that is not synced cannot validate.

Write `validator.json`:

```json
{
  "pubkey": PASTE_OUTPUT_OF_show-validator,
  "amount": "1000000uerth",
  "moniker": "<your-moniker>",
  "commission-rate": "0.1",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
```

For `pubkey`, paste the complete JSON object from `earthd comet show-validator`.
Paste it unquoted. It is an object, not a string.

```bash
earthd tx staking create-validator validator.json \
  --chain-id earth-1 --from <your-key> --gas auto --gas-adjustment 1.5
```

**Do not keep the consensus key on the node.** Use a remote signer. See
[Protecting the consensus key](./remote-signer.md). A remote signer fails closed:
if a signer is configured and none answers, the node signs nothing.

Double-signing causes slashing and a permanent tombstone. Never run two nodes
with the same consensus key. This applies during a migration and for any
duration.

---

## Upgrades

An upgrade halts the chain at an agreed height. The node stops and waits.

Use [cosmovisor](https://docs.cosmos.network/main/build/tooling/cosmovisor). It
stages the new binary in advance and changes to it automatically. Manual
replacement at the time of the upgrade height is not necessary.

The release notes of each upgrade give the name, the height, and the binary.

---

## Troubleshooting

**`expected chain id earth-1`** — the genesis file is wrong. Repeat step 3.

**Wrong app hash at a height** — the genesis file differs from the genesis file
of the network, or the binary is wrong. Check both.

**`set min gas price in app.toml`** — the node does not start until
`minimum-gas-prices` is set in `app.toml`. See step 4.

**No peers** — the seeds are wrong, or port 26656 does not accept inbound
connections. Peers must be able to dial this node.

**The node stops at a height and has peers** — usually an upgrade that this node
has not applied. Check the releases page.

**State sync stays at `Discovering snapshots`** — no peer offers a snapshot. The
source node requires a `snapshot-interval` that is not zero. A node cannot
produce a snapshot for a height that it has already passed.
