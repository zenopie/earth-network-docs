---
sidebar_position: 4
title: Protecting the consensus key
---

# Protecting the consensus key

Validating means running a node that is online constantly, which for most people
means renting one — a VPS, a dedicated box, a cloud instance, an Akash lease.
All of those have the same property: **someone else can read the disk.**

By default the consensus key sits at `$EARTH_HOME/config/priv_validator_key.json`
on that disk. Whoever operates the machine can copy it, and with it double-sign,
which on this chain costs 5% of stake and a permanent tombstone. That is not a
hypothetical about a malicious host; a backup snapshot, a support engineer, or a
provider compromise reaches the same file.

A remote signer splits the node from its key. The node keeps running on rented
hardware; the key lives on a machine you own. The node *asks* for signatures and
can no longer produce them.

```
home (dynamic IP, no open ports)          rented host
  tmkms ──dials──► tunnel ──────────►  earthd :26659
   key                                    no key
```

[tmkms](https://github.com/iqlusioninc/tmkms) is a separate program from
Iqlusion. It is not part of this chain and not shipped with it; you install and
run it yourself.

---

## Your home IP does not need to be static

tmkms is the *client*. CometBFT listens on `priv_validator_laddr` and the signer
dials in, so the machine holding the key never has to be reachable. Both
processes at home make outbound connections only — no port forwarding, no static
address, nothing to configure on a router.

The stable address is needed on the validator's side, which is the side that has
one.

## The reason people underrate this

tmkms records the last `(height, round, step)` it signed and refuses to sign at
or below it. That is protection against **equivocation**, not just theft. If the
node is compromised, or you accidentally run a second validator during a
migration, the signer will not produce the conflicting signature. It cannot sign
the thing that gets you slashed.

That state file must be durable and monotonic. Losing it, or restoring it from a
backup older than the chain's tip, is how a remote signer causes the exact fault
it exists to prevent — which is also why anything with ephemeral storage is a
poor place to run one. Prefer a boring always-on box with a real disk over a fast
one.

---

## Node side

Point CometBFT at a listener instead of a file:

```toml
# config.toml
priv_validator_laddr = "tcp://0.0.0.0:26659"
```

Unset, nothing changes and the node signs locally. Set, it **fails closed**: with
no signer answering, the node produces no blocks at all. Do not set it until the
signer is up and reachable.

Then keep 26659 off the public internet. How depends on your host — a firewall
rule, binding it to a private interface, or exposing it only to a tunnel
sidecar. Whatever the mechanism, it is not the thing that authenticates the
connection. The next section is why that distinction matters more than it looks.

## What the privval socket does not protect

Read this before deciding the port exposure is a detail.

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
rather than a nuisance. The double-sign guard blocks conflicting votes at or
below heights it has already seen; it cannot tell a forged future height from a
real one. It stops replays, not forgeries.

Authentication therefore has to come from the transport, because the protocol
does not supply it.

---

## Choosing a transport

Anything that authenticates both ends and carries a TCP stream will do. The
common options:

| | how it authenticates | notes |
| --- | --- | --- |
| WireGuard / Tailscale | peer keys | simplest if you can install it on the host |
| SSH tunnel | host + user keys | fine, but a dropped `ssh` stops signing silently |
| Cloudflare Tunnel + Access | service token | no inbound port on either side |

The rest of this page works through the Cloudflare option, because it needs no
open port on the node *or* at home. The security requirement is the same
whichever you pick: **something must reject a stranger who connects to 26659.**

### Cloudflare: publish the hostname

Add a public hostname of type **TCP** on your tunnel, pointing at the node:

```
signer.example.com -> tcp://node:26659
```

Address the node by *name* where you can — the tunnel daemon resolves it on its
own side, so a host whose address changes when it is rebuilt costs you nothing.
Pinning a private IP means pinning an address the provider assigns and may
reissue to someone else.

### Cloudflare: lock the hostname

**This is the control.** A public hostname is public. `cloudflared access tcp`
is an ordinary client and anyone can point one at your name, so until a policy
is attached, that name is a TCP endpoint onto the privval socket for whoever
guesses it.

Because both ends are machines, this wants a Cloudflare Access **service
token**, not an identity provider login:

1. Zero Trust → Access → **Service Auth** → create a service token. The Client
   Secret is shown **once**; it cannot be retrieved later, only replaced. Prefer
   a non-expiring token — an expiry date is a date your validator stops signing.
2. Zero Trust → Access → **Applications** → add a **Self-hosted** application for
   `signer.example.com`.
3. Give it one policy with action **Service Auth** — *not* Allow, which expects a
   human at an IdP and will lock out a headless signer — including that token.

Rotation is two-sided and ordered: issue the new token, add it to the policy,
restart the home-side client, *then* revoke the old one. Revoking first drops the
signer, and a dropped signer is a validator producing no blocks.

### Cloudflare: home side

```bash
export CF_ACCESS_CLIENT_ID=...access
export CF_ACCESS_CLIENT_SECRET=...

cloudflared access tcp --hostname signer.example.com --url localhost:26659
tmkms start -c tmkms.toml
```

cloudflared reads those two from the environment. Passing them as flags puts the
secret into `ps` output and your shell history.

Start the tunnel **first**. `cloudflared access tcp` binds `localhost:26659` and
forwards it, so tmkms dials the same address it would use for a local node —
which also means `tmkms.toml` is identical either way. Started in the other
order, tmkms dials a port nothing is listening on.

### Verify the lock before you trust it

From a machine with no token, this must be **refused**:

```bash
cloudflared access tcp --hostname signer.example.com --url localhost:26699
```

Do not skip it. An application that was never attached to the hostname behaves
identically to a working one from the authenticated side, so the failure is
invisible in exactly the direction that matters.

---

## tmkms configuration

The parts that are specific to this chain:

```toml
[[chain]]
id = "earth-1"
key_format = { type = "bech32", account_key_prefix = "earthpub", consensus_key_prefix = "earthvalconspub" }
state_file = "/path/to/state/earth-consensus.json"

[[providers.softsign]]
chain_ids = ["earth-1"]
key_type = "consensus"
path = "/path/to/secrets/earth-consensus.key"

[[validator]]
chain_id = "earth-1"
addr = "tcp://127.0.0.1:26659"
secret_key = "/path/to/secrets/kms-identity.key"
protocol_version = "v0.38"
reconnect = true
```

Three things that cost people an evening:

- **`protocol_version` must be `v0.38`** for CometBFT 0.38 / Cosmos SDK v0.53.
  `v0.34` is deprecated and becomes a hard error in a coming tmkms release.
- **The state file needs `round` as a *string*** and an explicit `block_id` key.
  Omit either and tmkms will not start.
- **Wrong bech32 prefixes do not stop signing.** The signature is over bytes, so
  it works — but every log line and every key you print comes out addressed to
  another chain, which is a bad way to audit a key.

Healthy output is a line per vote:

```
signed Proposal:8E3026E331 at h/r/s 6/0/0 (0 ms)
signed Prevote:8E3026E331  at h/r/s 6/0/1 (0 ms)
signed Precommit:8E3026E331 at h/r/s 6/0/2 (0 ms)
```

`Connection refused` in the seconds before the node is up is normal — tmkms
retries.

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
4. Start tmkms, then the node with `priv_validator_laddr` set.
5. Confirm blocks are being signed again.
6. **Delete the key from the rented host.** Skipping this leaves the key on the
   machine the whole exercise was about getting it off.
