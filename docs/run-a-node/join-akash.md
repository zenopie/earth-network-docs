---
sidebar_position: 3
title: Join on Akash
---

# Running a node on Akash

Everything in [Join the network](./join.md) still applies — same binary, same
genesis, same hash to check. This page is only the parts that differ when the
machine is rented rather than yours, and the part that matters most if you
intend to validate: **keeping the consensus key off it.**

If you only want a full node, read to the end of *Networking* and stop. The
signer sections are for validators.

---

## What is actually different

Three things, and they all come from the same fact: you do not own the host.

| | your own box | Akash |
| --- | --- | --- |
| storage | survives by default | survives only if declared `persistent` |
| address | fixed | provider ports change on every new lease |
| the key | on hardware you control | on hardware someone else controls |

The first two are configuration. The third is not, and it is the reason for
half this page.

---

## Before you start

An Akash deployment needs AKT in escrow, or a Console account with a card. You
also need somewhere to put the node's address, because a provider's external
ports are reassigned every time you recreate the lease — a tunnel or a proxy you
control, not a bare port.

---

## Storage: declare it persistent or lose the chain

A container's filesystem is discarded on every restart. The node's home holds
the genesis, the node key, the consensus key and the whole database, so a node
without persistent storage resyncs from height 1 on every reschedule — and a
*validator* without it comes back having forgotten what it signed.

```yaml
profiles:
  compute:
    node:
      resources:
        storage:
          - size: 8Gi                # root: the image
          - name: data
            size: 500Gi
            attributes:
              persistent: true
              class: beta3
```

Mount it at the node's home:

```yaml
services:
  node:
    params:
      storage:
        data:
          mount: /data
    env:
      - EARTH_HOME=/data
```

`class: beta3` rather than `beta2` — far more providers advertise it, and a
storage class nobody offers means no bids.

### Closing the lease destroys it

This is the one to internalise before you start. Closing a lease deletes its
persistent volumes: genesis, node key, **consensus key**, database. There is no
undo and no export.

So there are two kinds of change:

- **image and env** go in place with `PUT /v1/deployments/{dseq}`, volumes intact
- **endpoints and resources** are part of what the provider bid on. Changing them
  is rejected, and applying them means closing and recreating — which destroys
  the volume

Decide your ports and resources before you have state worth keeping. And if you
are on an Akash trial, deployments auto-close after 24 hours, which is the same
thing on a timer.

---

## Networking

`p2p` needs a real address. CometBFT otherwise advertises whatever address it
sees on itself — inside a container, an address nobody can dial — and hands that
to every peer it meets.

```yaml
    expose:
      - port: 26656
        as: 26656
        to:
          - global: true
```

Then read the external port back from the lease and tell the node about it:

```bash
earthd start --p2p.external-address <provider-host>:<external-port>
```

That port changes when you recreate the lease. If you publish a seed address,
publish one that does not move — a tunnel hostname or a proxy — rather than a
provider port.

**RPC and LCD are a different question.** Publishing them `global: true` puts an
unauthenticated RPC on the public internet attached to a validator. Put them
behind a tunnel, or do not publish them at all.

---

## Validators: get the key off the host

Everything above is ordinary operations. This is the part that is specific to
renting hardware.

By default the consensus key sits at `$EARTH_HOME/config/priv_validator_key.json`
on the provider's machine. Whoever operates that machine can read it, and with it
double-sign — which on this chain costs 5% of stake and a permanent tombstone.
Not a theoretical risk: it is someone else's disk.

A remote signer splits the node from its key. The node keeps running on rented
hardware; the key lives on a machine you own. The node *asks* for signatures and
can no longer produce them.

```
home (dynamic IP, no open ports)              Akash
  tmkms ──dials──► cloudflared ──tunnel──►  earthd :26659
   key                                       no key
```

[tmkms](https://github.com/iqlusioninc/tmkms) is a separate program from
Iqlusion. It is not part of this chain and not shipped with it; you install and
run it yourself.

### A dynamic home IP is fine

tmkms is the *client*. CometBFT listens on `priv_validator_laddr` and the signer
dials in, so the machine holding the key never has to be reachable. Both
processes at home make outbound connections only — no port forwarding, no static
address, nothing to configure on a router.

The stable hostname is needed on the validator's side, which is the side that has
one.

### The reason people underrate it

tmkms keeps the last `(height, round, step)` it signed and refuses to sign at or
below it. That is protection against **equivocation**, not just theft. If the
node is compromised, or you accidentally run a second validator during a
migration, the signer will not produce the conflicting signature. It cannot sign
the thing that gets you slashed.

That state file must be durable and monotonic. Losing it is how a remote signer
causes the exact fault it exists to prevent — which is also why an enclave with
ephemeral storage is a poor place to run one.

### Node side

```yaml
    env:
      - PRIV_VALIDATOR_LADDR=tcp://0.0.0.0:26659
```

Unset, nothing changes and the node signs locally.

Expose 26659 **to the tunnel service only** — never `global: true`:

```yaml
    expose:
      - port: 26659
        to:
          - service: cloudflared
```

Then a TCP public hostname on the tunnel: `signer.example.com -> tcp://node:26659`.

### What the privval socket does not protect

Read this before you decide the port exposure is a detail.

The connection is Secret Connection encrypted, but on this socket it is
effectively **unauthenticated in both directions**.

tmkms can pin the node's identity — `tcp://<node_id>@host:port` — but there is
nothing stable to pin, because CometBFT mints a throwaway key for the privval
listener on every process start:

```go
case "tcp":
  // TODO: persist this key so external signer can actually authenticate us
  listener = NewTCPListener(ln, ed25519.GenPrivKey())
```

Pin it anyway and the node dies on its next restart: tmkms rejects with
`validator peer ID mismatch`, the node gets `can't get pubkey: send: EOF` and
exits. So configure the address *without* the `<node_id>@` prefix and accept that
tmkms logs `unverified validator peer ID!` on every connect. That warning is
expected and cannot currently be cleared.

The node does not authenticate the KMS either — its listener accepts whoever
connects first.

The consequence: **anyone who can reach 26659 can impersonate the node to the KMS
and ask it to sign votes.** They cannot steal the key, but they can request
signatures at heights the real node has not reached, which is a double-sign risk
rather than a nuisance. tmkms's double-sign guard blocks conflicting votes at or
below heights it has already seen; it cannot tell a forged future height from a
real one.

Authentication therefore comes from the transport. That is why 26659 is exposed
only to the tunnel service. Do not shortcut it with a `global: true` port, even
temporarily.

### Home side

```bash
cloudflared access tcp --hostname signer.example.com --url localhost:26659
tmkms start -c tmkms.toml
```

tmkms points at `localhost:26659`; the tunnel carries it.

---

## Migrating a validator that is already running

Do this before anyone else bonds stake behind you. Moving a consensus key on a
live validator is the operation most likely to cause an accidental double-sign:
the window where both the old node and the new signer believe they may sign is
precisely the fault being guarded against.

1. Stop the validator. Confirm it is not producing blocks.
2. Copy `priv_validator_key.json` to the signer host and import it into tmkms.
3. Seed tmkms's state from `priv_validator_state.json`, so it does not start
   believing it has signed nothing.
4. Start tmkms, then the node with `PRIV_VALIDATOR_LADDR` set.
5. Confirm blocks are being signed again.
6. **Delete the key from the Akash host.** Skipping this leaves the key on the
   machine the whole exercise was about getting it off.

---

## Upgrades

See [Upgrades](./upgrades.md) for what a coordinated upgrade involves. The Akash
difference is only how the binary arrives: change the image digest and `PUT` the
deployment. The volume survives, so the node resumes at its stored height and
applies the upgrade handler.

Pin images by **digest**, not by tag. A tag can be repointed; a digest is the
bytes you tested.
