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

By default the consensus key sits at `$EARTH_HOME/config/priv_validator_key.json`
— on the provider's disk. Whoever operates that machine can read it, and with it
double-sign, which on this chain costs 5% of stake and a permanent tombstone.

This is not an Akash problem; it is true of any host you rent. The fix is a
remote signer, and it has its own page: **[Protecting the consensus
key](./remote-signer.md)**.

The one Akash-specific part is exposing the signer port to the tunnel service
rather than to the internet:

```yaml
    expose:
      - port: 26659
        to:
          - service: cloudflared
```

That removes the provider port. It is not what authenticates the connection —
see the signer page, which is emphatic about the difference.

---

## Upgrades

See [Upgrades](./upgrades.md) for what a coordinated upgrade involves. The Akash
difference is only how the binary arrives: change the image digest and `PUT` the
deployment. The volume survives, so the node resumes at its stored height and
applies the upgrade handler.

Pin images by **digest**, not by tag. A tag can be repointed; a digest is the
bytes you tested.
