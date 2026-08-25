---
sidebar_position: 4
title: Upgrades
---

# Upgrading the chain

How a coordinated upgrade runs, and the things that go wrong.

Rehearsed end to end on a local chain — `scripts/rehearse-upgrade.sh` reproduces
it. Everything below has been observed, not assumed.

---

## What happens

1. A `MsgSoftwareUpgrade` proposal names an upgrade and a **height**.
2. The proposal passes and the plan is stored.
3. Every node reaches that height and **halts**, logging:

   ```
   UPGRADE "<name>" NEEDED at height: <height>
   ```

4. Operators swap in a binary whose `Upgrades` list contains a matching entry.
5. The binary runs the handler once, migrates state, and continues.

A node running a binary **without** a matching entry halts again at the same
height. That is the intended behaviour and it is the thing to expect at 3am: the
chain is not broken, the binary is wrong.

---

## The mistake that costs a week

**The plan height must be beyond the end of the voting period, not beyond now.**

`x/upgrade` rejects a plan whose height has already passed by the time the
proposal executes. The proposal then shows `PROPOSAL_STATUS_FAILED` — it passed
the vote and failed to apply, which reads confusingly because "passed" and
"failed" are both true of it.

The voting period is **7 days**. At 5s blocks that is roughly **120,000 blocks**.
So a real proposal has to target a height at least that far out, plus margin for
block times drifting.

```
current height + (voting period ÷ block time) + margin
```

Get it wrong and you wait another 7 days to propose again.

This is not hypothetical: the first run of the rehearsal script failed exactly
this way, with a plan 25 blocks out and a voting period 40 blocks long.

---

## Writing the upgrade

Add an entry to `Upgrades` in `app/upgrades.go`:

```go
var Upgrades = []Upgrade{
    {Name: "v2", CreateHandler: defaultUpgradeHandler},
}
```

`defaultUpgradeHandler` runs the registered module migrations and nothing else,
which is right for an upgrade that changes logic or parameters but not the set of
modules.

**If the upgrade adds, renames or removes a module store**, declare it —
otherwise the node fails to start, because the commit multistore has to be told
about a new store key before it can load it:

```go
{
    Name: "v2",
    CreateHandler: defaultUpgradeHandler,
    StoreUpgrades: storetypes.StoreUpgrades{Added: []string{"newmodule"}},
}
```

Entries stay in the list after they have been applied. A node syncing from
genesis replays each in turn, so removing one breaks sync from scratch.

---

## Proposing it

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
  "summary": "What changes, and why. Link the release."
}
```

### The `info` field

`info` is not free text. Cosmovisor parses it as JSON and will refuse a plan it
cannot read, so get this shape right:

```json
{
  "binaries": {
    "linux/amd64": "https://github.com/zenopie/earth-network-chain/releases/download/v0.5.0/earthd_v0.5.0_linux_amd64.tar.gz?checksum=sha256:PUT_THE_REAL_HASH_HERE",
    "linux/arm64": "https://github.com/zenopie/earth-network-chain/releases/download/v0.5.0/earthd_v0.5.0_linux_arm64.tar.gz?checksum=sha256:PUT_THE_REAL_HASH_HERE"
  }
}
```

The keys are `GOOS/GOARCH`; a node that finds no key for its own platform (and no
`any` key) fails the download. The hashes are the ones in `checksums.txt` on the
release.

**The `?checksum=` is mandatory**, not decoration. It is the only thing binding
what a node downloads and executes to what governance actually approved, and
nodes run with `DAEMON_DOWNLOAD_MUST_HAVE_CHECKSUM=true` will reject a plan
without it. Operators doing a manual upgrade read the same hashes to verify by
hand.

```bash
earthd tx gov submit-proposal plan.json --from <key> \
  --chain-id earth-1 --gas auto --gas-adjustment 1.5 --gas-prices 0.005uerth
```

**Every transaction needs `--gas-prices`** at or above the node's minimum, or it
is rejected with `insufficient fee` before it reaches the mempool.

---

## On the day

At the upgrade height every node stops on purpose and refuses to continue on the
old binary. That is the chain protecting itself: continuing would mean two
versions producing different state from the same blocks.

Someone or something has to put the new binary in place. Two ways to do that,
below. **Both are supported** — cosmovisor is a convenience, not a requirement,
and a manual upgrade is not a worse upgrade.

Whichever you use, the release tarball carries `bin/earthd` and a `lib/` holding
`libwasmvm` and the C++ runtime. `earthd` links `libwasmvm` dynamically and the
two are version-locked, so **never pair one release's `earthd` with another
release's `libwasmvm`** — keep `bin/` and `lib/` together.

---

### Path A — manual

Nothing to install ahead of time. You need to be present at the height.

**Before the height**, download and verify, but do not install yet:

```bash
VERSION=v0.5.0
ARCH=amd64

curl -LO https://github.com/zenopie/earth-network-chain/releases/download/$VERSION/earthd_${VERSION}_linux_${ARCH}.tar.gz
curl -LO https://github.com/zenopie/earth-network-chain/releases/download/$VERSION/checksums.txt
sha256sum -c checksums.txt --ignore-missing        # must say OK

mkdir -p ~/earthd-$VERSION
tar -C ~/earthd-$VERSION -xzf earthd_${VERSION}_linux_${ARCH}.tar.gz
~/earthd-$VERSION/bin/earthd version --long        # confirm it is the right build
```

Keep the current binary. If the upgrade goes wrong it is what you roll back to.

**At the height** the node halts, logging `UPGRADE "<name>" NEEDED at height`.
Then swap both halves and restart:

```bash
sudo systemctl stop earthd                          # if it has not already exited

sudo install -m755 ~/earthd-$VERSION/bin/earthd /usr/local/bin/
sudo install -m644 ~/earthd-$VERSION/lib/*      /usr/local/lib/

earthd version --long                               # must report $VERSION
sudo systemctl start earthd
```

The log shows `applying upgrade "<name>" at height` and blocks resume.

Installing to `/usr/local/bin` and `/usr/local/lib` works with no `ldconfig` and
no `LD_LIBRARY_PATH`: the binary looks for its libraries at `../lib` relative to
itself.

---

### Path B — cosmovisor

[Cosmovisor](https://docs.cosmos.network/main/build/tooling/cosmovisor) is a
small supervisor that runs `earthd` for you. It watches for the halt, puts the
new binary in place and restarts the node — so an upgrade at 03:00 does not need
you awake at 03:00.

It is a separate program. It is not part of `earthd`, and running it is optional.

**One-time setup:**

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.1

export DAEMON_NAME=earthd
export DAEMON_HOME=$HOME/.earth        # your node home, NOT a temp directory

mkdir -p $DAEMON_HOME/cosmovisor/genesis/bin
cp $(which earthd) $DAEMON_HOME/cosmovisor/genesis/bin/
```

Then run the node through it — `cosmovisor run` takes the same arguments you
would give `earthd`:

```bash
cosmovisor run start --home $DAEMON_HOME
```

In a systemd unit, set the environment there:

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

`DAEMON_HOME` must be on storage that survives a restart. Cosmovisor keeps
downloaded binaries under `$DAEMON_HOME/cosmovisor/`, so on a host with an
ephemeral filesystem it would fetch an upgrade, restart, and find it gone.

**Then pick how it gets the binary — download it, or stage it yourself.**

**B1: let it download.** With `DAEMON_ALLOW_DOWNLOAD_BINARIES=true`, cosmovisor
reads the URL from the proposal's `info` field, verifies the `?checksum=`, unpacks
it and restarts. Nothing to do before the height. This is the unattended option,
and it is why `?checksum=` is mandatory: keep
`DAEMON_DOWNLOAD_MUST_HAVE_CHECKSUM=true` so a plan without one is refused rather
than trusted.

**B2: stage it yourself.** Set `DAEMON_ALLOW_DOWNLOAD_BINARIES=false` if you would
rather nothing be fetched from the internet at an upgrade height. Put the release
under the upgrade name **before** the height — the name must match the plan's
`name` exactly:

```bash
VERSION=v0.5.0
UPGRADE_NAME=v0.5.0                     # the plan's `name`, which need not equal $VERSION

mkdir -p $DAEMON_HOME/cosmovisor/upgrades/$UPGRADE_NAME
tar -C $DAEMON_HOME/cosmovisor/upgrades/$UPGRADE_NAME \
    -xzf earthd_${VERSION}_linux_amd64.tar.gz
```

That gives `upgrades/$UPGRADE_NAME/bin/earthd` with its own `lib/` beside it,
which is exactly the layout cosmovisor expects and the same one it would have
produced by downloading.

**Verify the staging before the height, not during it:**

```bash
$DAEMON_HOME/cosmovisor/upgrades/$UPGRADE_NAME/bin/earthd version --long
```

If that prints the new version, the swap will work. If it errors, you have found
out while the chain is still producing blocks.

---

### Which to use

| | Manual | Cosmovisor |
|---|---|---|
| Setup beforehand | none | one-time |
| Present at the height | **yes** | no |
| Downloads at the height | no | B1 yes, B2 no |
| Fewest moving parts | ✅ | |

A single node you watch is fine manually. Cosmovisor earns its place when the
height lands at an inconvenient hour, or when you run more than one node.

---

## If it goes wrong

**Halts again after swapping** — the binary has no entry matching the plan name.
Check `earthd version` and the spelling of the name.

**Fails to start with a store error** — the upgrade adds a module store and the
entry has no `StoreUpgrades`.

**The upgrade was a mistake and has to be skipped.** Every node must agree, or
those that skip fork from those that do not:

```bash
earthd start --unsafe-skip-upgrades <height>
```

Coordinate it publicly and explicitly. A node that skips alone is on a different
chain from that block on.

**The chain is halted and consensus cannot resume.** That is a genesis restart,
not an upgrade: export state, build a new genesis from it, and start a new chain
id. `scripts/build-genesis.sh` and `networks/genesis/README.md` cover the mechanics.

---

## Rehearsing

Two scripts, one per path above. Both start a throwaway chain, pass a real
proposal and wait for a real halt — nothing is mocked. Both edit
`app/upgrades.go` while running and restore it on exit, including on failure.

**The manual path:**

```bash
scripts/rehearse-upgrade.sh
```

Passes a proposal, waits for the halt, checks the old binary **refuses to
continue**, rebuilds with a handler, and checks the chain resumes past the halt
height. That refusal is the important assertion: a binary that continued past an
upgrade height without a handler would be a consensus break.

**The cosmovisor path:**

```bash
scripts/rehearse-cosmovisor.sh
```

Serves a new binary over loopback, puts its URL and sha256 in the plan's `info`,
and then does nothing — cosmovisor has to halt, download, verify, unpack, swap
and restart on its own. It checks the downloaded binary matches what was served
and differs from the old one, so a chain that simply carried on cannot pass.

It needs no release to exist and works offline, and it exercises the three things
that only fail at an upgrade height: the `info` JSON shape, the archive layout,
and checksum enforcement.

Run the one matching how you actually upgrade, before any upgrade that matters.
