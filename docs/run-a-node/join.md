---
sidebar_position: 2
title: Join the network
---

# Running an Earth node

This page contains the full procedure to sync a node on `earth-1`.

If you cannot sync a node with this page alone, that is a defect in this page.
Open an issue.

{/* TODO(relaunch): fill in the launch tag, genesis time, genesis sha256 and
the validator's node id below once networks/genesis.json is final. Confirm the
chain id: networks/genesis/chain.json still says earth-1. */}

> **The network relaunches from a new genesis** with private ERTH, ANML and
> staking. Nothing from the earlier `earth-1` carries over: no balances, no
> registrations, no history. The launch tag, the genesis time and the genesis
> checksum are published in the release notes and on this page before launch.
> It starts with one validator, so a node joining syncs from that one peer and
> adds the second.

## Which binary

A node runs the binary for the part of the chain it is processing. Each upgrade
below halted the chain at its height, and the next binary continued from there.

| Upgrade | Height | Binary from that height |
| --- | --- | --- |
| launch | 1 | the launch tag |

- **State sync** (section 4b) starts near the tip, so you need only the binary
  for the current height: the last row that has already happened.
- **Replaying from genesis** needs every binary in turn. Start on the launch tag
  under cosmovisor with download enabled, and it fetches each later binary at its
  height. See [Upgrades](./upgrades.md#method-b--cosmovisor).

---

## 1. Get the binary

Download from the [latest release](https://github.com/zenopie/earth-network-chain/releases/latest):

```bash
VERSION=<launch-tag>    # see "Which binary" above
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
<published with the launch release>  genesis.json
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
persistent_peers = "<validator-node-id>@<host>:<port>"
# The address that other nodes use to reach this node. Set it if the node is
# behind NAT, in a container, or at a provider that maps ports. If it is unset,
# CometBFT advertises the address that it observes on itself and gives that
# address to each peer. Other nodes then cannot dial this node.
external_address = "your.host.or.ip:26656"
```

Port **26656** must accept inbound connections. A node without inbound
connectivity can still sync, because it dials out. But no peer can dial it. It
therefore adds no connectivity to the network and cannot serve state sync.

**The validator's public P2P address is published here at launch.** Its node
id is what `https://rpc.erth.network/status` reports under `node_info.id`. The
host and port are assigned by its hosting provider once the launch lease runs.
Until this page gives the full address, ask in the project's channels for a
peer.

For the Docker image, use the `SEEDS`, `PERSISTENT_PEERS`, and `EXTERNAL_ADDRESS`
environment variables. The entrypoint writes them into `config.toml` at each
start, so a restart applies a change.

**Minimum gas price**, in `~/.earth/config/app.toml`. This value is **required.
The node does not start without it.** The error message does not name the file:

```
set min gas price in app.toml or flag or env variable
```

The node does not relay a transaction below this value. This is a per-node
setting, not a chain rule. For ordinary signed transactions there is no fee
module, so the effective floor of the network is the value that most validators
select:

```toml
minimum-gas-prices = "0.005uerth"
```

Fees are paid in ERTH only. Do not list `uanml`: ANML exists only in the
shielded pool and the chain refuses it as a fee. Private transactions pay their
fee from a shielded note, and this node checks that fee against the same
minimum when it admits them. Private transactions also have a **consensus**
floor: the `x/shielded` parameter `min_fee`, 1,000 uerth (0.001 ERTH) at
genesis. Every node enforces it in blocks too, whatever its own setting, and
only governance can change it.

**Mempool**, in `app.toml`. This value is **required** on this chain:

```toml
[mempool]
# Earth's private transactions carry no signer. The SDK's priority and
# sender-nonce mempools key transactions by signer and sequence and reject a
# transaction with no signer outright, so any value other than -1 makes this
# node drop every private transaction: claims, votes, transfers, stake.
max-txs = -1
```

`-1` is the default. Check it was not changed by a config template you copied
from another chain. A node with any other value still follows blocks, but
refuses private transactions sent to it, and a validator with any other value
never proposes them.
**Pruning**, in `app.toml`. Select by the role of the node:

| Role | Setting |
| --- | --- |
| Validator | `pruning = "default"` |
| Public RPC | `pruning = "custom"`, `pruning-keep-recent = "362880"`, `pruning-interval = "100"` |

**The official node is a full-history node.** The project runs one node: the
validator behind `rpc.erth.network` and `lcd.erth.network`. It has kept every
block, every block's results and every state since block 1, and it is the node
the official privacy indexer reads. You do not need your own full-history node
to look something up at an old height.

**Running your own privacy indexer?** Its node must hold every block and its
results from genesis. Wallets download every note ever created, so an index
with a gap is useless. Start it from genesis, never by state sync, and set:

```toml
# app.toml
pruning = "nothing"
min-retain-blocks = 0

# config.toml
[storage]
discard_abci_responses = false   # the indexer reads block_results

[tx_index]
indexer = "kv"
```

Its disk only grows, so size it well past the Public RPC column under
[Hardware](#hardware) and watch it. Under cosmovisor, set
`UNSAFE_SKIP_BACKUP=true`: the pre-upgrade backup copies all of `data/`, and on
a node like this it can fill the disk at an upgrade height.

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

`enabled-unsafe-cors` lets any website read the node and broadcast through it.
The official node turns it on, because it is both the validator and the public
LCD that the web app calls from the browser. That is a deliberate choice for a
single-node network. If you run your own validator, leave it off and serve
browsers from a separate read-only node.

---

## 4b. State sync (optional, and much faster)

State sync fetches state at a recent height from a peer. It does not replay
each block.

**This chain gains more from state sync than most chains.** A replay re-executes
the transactions of each block, and each passport registration and each private
transaction verifies a zero-knowledge proof. A chain that only transfers tokens
replays quickly. This chain re-runs every proof, so replay cost increases with
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

| | Validator | Public RPC |
| --- | --- | --- |
| CPU | 4 cores | 4 cores |
| RAM | 16 GB | 16 GB |
| Disk | 500 GB SSD | 1 TB SSD |

Use an SSD. Do not use a rotating disk. The node calls fsync at each block.

**Two requirements are specific to this chain.**

- Each passport registration and each private transaction verifies a
  zero-knowledge proof on-chain. This uses significant CPU time and the node
  cannot omit it. Allocate more CPU capacity than a chain of this size usually
  requires.
- The CPU must support the **ADX** instruction set (Intel Broadwell or later,
  AMD Zen or later). The proof verifier is built with it, and on an older CPU
  `earthd` exits with `Illegal instruction`, even for `earthd version`. On a
  cloud or Akash host, check `grep -c adx /proc/cpuinfo` is not zero before you
  commit to it.

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

**Your stake on Earth is your self-bond and the private stake module.**
Ordinary delegation does not exist here. The chain refuses `delegate`,
`unbond` and `cancel-unbond` from any account except a validator's own operator
account bonding to itself, and it refuses `redelegate` from every account,
operators included. Everyone else stakes privately through `x/shieldedstaking`,
which is the only delegator besides validators themselves. Its delegations to
you change once a day, at the end of an epoch.

- To add or remove your own stake, use `earthd tx staking delegate` or
  `unbond` from your operator account, to your own validator. A self-bond
  cannot be moved to another validator; unbond it instead.
- Your self-bond, your commission and your governance votes are public. Being a
  validator is a public role.
- Your vote on a proposal also covers the private stake delegated to you that
  does not vote itself. A private staker who votes takes their weight out of
  yours.
- **Your income is restaked, not paid out.** The chain sets your operator's
  withdraw address to a reward escrow that only it controls. At each epoch end
  your commission and your self-bond rewards are withdrawn into the escrow and
  delegated again as self-bond. You cannot withdraw them: the chain refuses
  an operator's `MsgWithdrawDelegatorReward`, every
  `MsgWithdrawValidatorCommission` and `MsgSetWithdrawAddress`, also through
  authz, governance or a contract. The only way to take income out is to unbond
  your self-bond, which takes the unbonding time. Compounding skips a validator
  while it is jailed or unbonded. When a validator is removed, its escrow is
  released to its operator.
- The private stakers' rewards are restaked for them once a day, the same
  way.

Fund the account with transparent ERTH. ERTH you receive privately has to be
unshielded to your operator address first, which makes that amount public.
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

**Private transactions never reach the chain through this node** — check
`max-txs = -1` under `[mempool]` in `app.toml`. See step 4.

**`Illegal instruction` on any `earthd` command** — the CPU lacks ADX. See
Hardware.

**No peers** — the seeds are wrong, or port 26656 does not accept inbound
connections. Peers must be able to dial this node.

**The node stops at a height and has peers** — usually an upgrade that this node
has not applied. Check the releases page.

**State sync stays at `Discovering snapshots`** — no peer offers a snapshot. The
source node requires a `snapshot-interval` that is not zero. A node cannot
produce a snapshot for a height that it has already passed.
