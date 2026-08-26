---
sidebar_position: 4
title: Upgrades
---

# Upgrading the chain

This page describes the procedure for a coordinated upgrade and its failure
modes.

The procedure was rehearsed completely on a local chain.
`scripts/rehearse-upgrade.sh` repeats it. All statements below are observed
results.

---

## Sequence of events

1. A `MsgSoftwareUpgrade` proposal names an upgrade and a **height**.
2. The proposal passes. The chain stores the plan.
3. Each node reaches that height and **halts**. It logs:

   ```
   UPGRADE "<name>" NEEDED at height: <height>
   ```

4. Operators install a binary whose `Upgrades` list contains a matching entry.
5. The binary runs the handler one time, migrates state, and continues.

A binary **without** a matching entry halts again at the same height. This is
the correct behaviour. If it occurs, the chain is not defective. The binary is
wrong.

---

## The most expensive error

**Set the plan height after the end of the voting period, not after the current
height.**

`x/upgrade` rejects a plan if its height has passed at the time of execution.
The proposal then reports `PROPOSAL_STATUS_FAILED`. The proposal passed the vote
and failed to apply. Both statements are correct, which makes the status
difficult to interpret.

The voting period is **7 days**. At 5-second blocks, this is approximately
**120,000 blocks**. A proposal must therefore target a height at least that
distance ahead, with margin for variation in block time.

```
current height + (voting period ÷ block time) + margin
```

An error here costs another 7 days before you can propose again.

This is a recorded failure, not a theoretical one. The first run of the
rehearsal script failed in this way. The plan was 25 blocks ahead and the voting
period was 40 blocks.

---

## Writing the upgrade

Add an entry to `Upgrades` in `app/upgrades.go`:

```go
var Upgrades = []Upgrade{
    {Name: "v2", CreateHandler: defaultUpgradeHandler},
}
```

`defaultUpgradeHandler` runs the registered module migrations and no other
operation. Use it for an upgrade that changes logic or parameters but does not
change the set of modules.

**If the upgrade adds, renames, or removes a module store, declare it.** The
commit multistore requires the new store key before it can load the store. If
you do not declare it, the node does not start:

```go
{
    Name: "v2",
    CreateHandler: defaultUpgradeHandler,
    StoreUpgrades: storetypes.StoreUpgrades{Added: []string{"newmodule"}},
}
```

Keep entries in the list after the chain applies them. A node that syncs from
genesis replays each entry in sequence. Removal of an entry stops a sync from
genesis.

---

## Submitting the proposal

```bash
CURRENT=$(earthd query block --type height 0 -o json | jq -r .header.height)
HEIGHT=$(( CURRENT + 130000 ))   # 7-day vote at 5s blocks, plus margin
GOV=$(earthd query auth module-account gov -o json | jq -r .account.value.address)
```

`plan.json`:

```json
{
  "messages": [
    {
      "@type": "/cosmos.upgrade.v1beta1.MsgSoftwareUpgrade",
      "authority": "GOV_ADDRESS",
      "plan": { "name": "v2", "height": "HEIGHT", "info": "release URL and sha256" }
    }
  ],
  "metadata": "",
  "deposit": "1000000uerth",
  "title": "Upgrade to v2",
  "summary": "The changes and the reason for them. Include the release link."
}
```

### The `info` field

`info` is not free text. Cosmovisor parses it as JSON and rejects a plan that it
cannot parse. Use this structure:

```json
{
  "binaries": {
    "linux/amd64": "https://github.com/zenopie/earth-network-chain/releases/download/v0.5.0/earthd_v0.5.0_linux_amd64.tar.gz?checksum=sha256:PUT_THE_REAL_HASH_HERE",
    "linux/arm64": "https://github.com/zenopie/earth-network-chain/releases/download/v0.5.0/earthd_v0.5.0_linux_arm64.tar.gz?checksum=sha256:PUT_THE_REAL_HASH_HERE"
  }
}
```

The keys are `GOOS/GOARCH`. A node that finds no key for its platform, and no
`any` key, cannot download the binary. Use the hashes from `checksums.txt` in
the release.

**The `?checksum=` parameter is mandatory.** It is the only link between the
binary that a node downloads and executes and the binary that governance
approved. A node with `DAEMON_DOWNLOAD_MUST_HAVE_CHECKSUM=true` rejects a plan
without it. Operators who upgrade manually use the same hashes to verify the
download.

```bash
earthd tx gov submit-proposal plan.json --from <key> \
  --chain-id earth-1 --gas auto --gas-adjustment 1.5 --gas-prices 0.005uerth
```

**Each transaction requires `--gas-prices`** at or above the minimum of the
node. Without it, the node rejects the transaction with `insufficient fee`
before the mempool receives it.

---

## At the upgrade height

At the upgrade height, each node stops and does not continue on the old binary.
This is a protection. If the node continued, two versions would produce
different state from the same blocks.

An operator or a supervisor must install the new binary. The two methods are
below. **Both are supported.** Cosmovisor is a convenience, not a requirement. A
manual upgrade is not inferior.

For both methods, the release tarball contains `bin/earthd` and a `lib/`
directory with `libwasmvm` and the C++ runtime. `earthd` links `libwasmvm`
dynamically and the two are version-locked. **Never use the `earthd` of one
release with the `libwasmvm` of another release.** Keep `bin/` and `lib/`
together.

---

### Method A — manual

This method requires no preparation on the host. It requires an operator at the
upgrade height.

**Before the height**, download and verify the release. Do not install it yet:

```bash
VERSION=v0.5.0
ARCH=amd64

curl -LO https://github.com/zenopie/earth-network-chain/releases/download/$VERSION/earthd_${VERSION}_linux_${ARCH}.tar.gz
curl -LO https://github.com/zenopie/earth-network-chain/releases/download/$VERSION/checksums.txt
sha256sum -c checksums.txt --ignore-missing        # the result must be OK

mkdir -p ~/earthd-$VERSION
tar -C ~/earthd-$VERSION -xzf earthd_${VERSION}_linux_${ARCH}.tar.gz
~/earthd-$VERSION/bin/earthd version --long        # confirm the build
```

Keep the current binary. It is the rollback target if the upgrade fails.

**At the height**, the node halts and logs `UPGRADE "<name>" NEEDED at height`.
Install both parts and restart:

```bash
sudo systemctl stop earthd                          # if it has not exited

sudo install -m755 ~/earthd-$VERSION/bin/earthd /usr/local/bin/
sudo install -m644 ~/earthd-$VERSION/lib/*      /usr/local/lib/

earthd version --long                               # must report $VERSION
sudo systemctl start earthd
```

The log then shows `applying upgrade "<name>" at height`, and block production
continues.

Installation to `/usr/local/bin` and `/usr/local/lib` needs no `ldconfig` and no
`LD_LIBRARY_PATH`. The binary looks for its libraries at `../lib`, relative to
its own location.

---

### Method B — cosmovisor

[Cosmovisor](https://docs.cosmos.network/main/build/tooling/cosmovisor) is a
supervisor that runs `earthd`. It detects the halt, installs the new binary, and
restarts the node. An operator does not need to be present at the upgrade
height.

Cosmovisor is a separate program. It is not part of `earthd` and its use is
optional.

**Initial setup:**

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.1

export DAEMON_NAME=earthd
export DAEMON_HOME=$HOME/.earth        # the node home, not a temporary directory

mkdir -p $DAEMON_HOME/cosmovisor/genesis/bin
cp $(which earthd) $DAEMON_HOME/cosmovisor/genesis/bin/
```

Then run the node through cosmovisor. `cosmovisor run` accepts the same
arguments as `earthd`:

```bash
cosmovisor run start --home $DAEMON_HOME
```

In a systemd unit, set the environment in the unit file:

```ini
[Service]
Environment="DAEMON_NAME=earthd"
Environment="DAEMON_HOME=/home/earth/.earth"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
Environment="DAEMON_DOWNLOAD_MUST_HAVE_CHECKSUM=true"
ExecStart=/home/earth/go/bin/cosmovisor run start --home /home/earth/.earth
Restart=always
```

`DAEMON_HOME` must use storage that survives a restart. Cosmovisor stores
downloaded binaries in `$DAEMON_HOME/cosmovisor/`. On a host with an ephemeral
filesystem, it downloads an upgrade, restarts, and then cannot find the binary.

**Then select how cosmovisor obtains the binary: by download, or from a staged
copy.**

**B1: download.** With `DAEMON_ALLOW_DOWNLOAD_BINARIES=true`, cosmovisor reads
the URL from the `info` field of the proposal, verifies the `?checksum=` value,
extracts the archive, and restarts. No action is required before the height.
This is the unattended method. Keep `DAEMON_DOWNLOAD_MUST_HAVE_CHECKSUM=true`,
so that cosmovisor rejects a plan without a checksum.

**B2: staged copy.** Set `DAEMON_ALLOW_DOWNLOAD_BINARIES=false` to prevent any
download at the upgrade height. Extract the release under the upgrade name
**before** the height. The directory name must match the `name` field of the
plan exactly:

```bash
VERSION=v0.5.0
UPGRADE_NAME=v0.5.0                     # the plan `name`, which can differ from $VERSION

mkdir -p $DAEMON_HOME/cosmovisor/upgrades/$UPGRADE_NAME
tar -C $DAEMON_HOME/cosmovisor/upgrades/$UPGRADE_NAME \
    -xzf earthd_${VERSION}_linux_amd64.tar.gz
```

The result is `upgrades/$UPGRADE_NAME/bin/earthd` with its `lib/` directory
beside it. This is the layout that cosmovisor expects and the layout that a
download produces.

**Verify the staged copy before the height:**

```bash
$DAEMON_HOME/cosmovisor/upgrades/$UPGRADE_NAME/bin/earthd version --long
```

If this prints the new version, the change will succeed. If it reports an error,
you have found the problem while the chain still produces blocks.

---

### Comparison of the methods

| | Manual | Cosmovisor |
|---|---|---|
| Preparation on the host | None | One time |
| Operator present at the height | **Required** | Not required |
| Download at the height | No | B1 yes, B2 no |
| Number of components | Lower | Higher |

For a single node with an operator present, use the manual method. Cosmovisor is
better when the upgrade height falls at an inconvenient time, or when one
operator runs several nodes.

---

## Failure modes

**The node halts again after the change** — the binary has no entry that matches
the plan name. Check `earthd version` and the spelling of the name.

**The node does not start and reports a store error** — the upgrade adds a
module store and the entry has no `StoreUpgrades`.

**The upgrade is incorrect and must be skipped.** Each node must skip it. Nodes
that skip fork from nodes that do not:

```bash
earthd start --unsafe-skip-upgrades <height>
```

Coordinate this publicly. A node that skips alone is on a different chain from
that block onward.

**The chain is halted and consensus cannot continue.** This requires a genesis
restart, not an upgrade. Export the state, build a new genesis file from it, and
start a new chain id. See `scripts/build-genesis.sh` and
`networks/genesis/README.md`.

---

## Rehearsal

There are two scripts, one for each method above. Both start a temporary chain,
pass an actual proposal, and wait for an actual halt. No component is mocked.
Both scripts edit `app/upgrades.go` during the run and restore it on exit,
including after a failure.

**The manual method:**

```bash
scripts/rehearse-upgrade.sh
```

This script passes a proposal, waits for the halt, confirms that the old binary
**refuses to continue**, rebuilds with a handler, and confirms that the chain
continues past the halt height. The refusal is the important check. A binary
that continued past an upgrade height without a handler would break consensus.

**The cosmovisor method:**

```bash
scripts/rehearse-cosmovisor.sh
```

This script serves a new binary over loopback and puts its URL and sha256 in the
`info` field of the plan. It then performs no further action. Cosmovisor must
halt, download, verify, extract, install, and restart without assistance. The
script confirms that the downloaded binary matches the served binary and differs
from the old binary. A chain that continued without an upgrade cannot pass this
check.

The script requires no published release and works offline. It exercises the
three items that fail only at an upgrade height: the `info` JSON structure, the
archive layout, and checksum enforcement.

Run the script for your upgrade method before any upgrade of consequence.
