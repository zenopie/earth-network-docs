---
sidebar_position: 3
title: Protecting the consensus key
---

# Protecting the consensus key

A validator must run continuously. Most operators therefore rent the machine.
The operator of that machine can read its disk.

By default the consensus key is a file on that disk:

```
$EARTH_HOME/config/priv_validator_key.json
```

Anyone who copies this file can double-sign. On this chain, double-signing costs
5% of stake and a permanent tombstone.

A remote signer separates the node from the key. The node runs on the rented
host. The key stays on a host you own. The node requests signatures. It cannot
create them.

```
your host                              rented host
  tmkms ──dials──► tunnel ─────────► earthd :26659
   key                                  no key
```

[tmkms](https://github.com/iqlusioninc/tmkms) is a separate program from
Iqlusion. Install and run it yourself.

---

## Two properties to know first

**The signer host does not need a fixed address.** tmkms is the client.
CometBFT listens on `priv_validator_laddr`. tmkms dials in. Both processes on
your side make outbound connections only. No port forwarding is required.

**tmkms also prevents double-signing.** It stores the last signed
`(height, round, step)`. It refuses to sign at or below that value. If the node
host is compromised, or if you start a second validator, tmkms does not create
the conflicting signature.

This state file must be durable. Do not delete it. Do not restore it from a
backup that is older than the chain tip. Both actions cause the fault that the
signer prevents. Do not run a signer on a host with ephemeral storage.

---

## Node host

Set the listener in `config.toml`:

```toml
priv_validator_laddr = "tcp://0.0.0.0:26659"
```

If unset, the node signs locally.

If set, the node fails closed. With no signer connected, the node produces no
blocks. Configure the signer first. Set this value last.

Then block public access to port 26659. Use a firewall rule, a private
interface, or a tunnel. This step limits who can reach the port. It does not
authenticate the connection. Read the next section.

---

## Limits of the privval socket

The connection uses Secret Connection encryption. On this socket, it is
unauthenticated in both directions.

tmkms can pin the node identity with `tcp://<node_id>@host:port`. There is no
stable identity to pin. CometBFT creates a new key for the privval listener at
every start:

```go
case "tcp":
  // TODO: persist this key so external signer can actually authenticate us
  listener = NewTCPListener(ln, ed25519.GenPrivKey())
```

If you pin the identity, the node fails at its next restart. tmkms reports
`validator peer ID mismatch`. The node reports `can't get pubkey: send: EOF` and
exits.

Configure the address without the `<node_id>@` prefix. tmkms then logs
`unverified validator peer ID!` at every connection. This warning is expected.
You cannot clear it.

The node does not authenticate the signer either. The listener accepts the first
client that connects.

Result: **any client that reaches port 26659 can request signatures.** The client
cannot read the key. It can request signatures for heights that the node has not
reached. This is a double-signing risk.

The tmkms guard does not prevent this. It compares each request against the last
signed height. It cannot identify a forged future height. It blocks replays. It
does not block forgeries.

The transport must therefore authenticate the client.

---

## Transport options

Any transport that authenticates both ends and carries TCP is acceptable.

| Transport | Authentication | Note |
| --- | --- | --- |
| WireGuard or Tailscale | peer keys | Simplest if you can install it on the node host |
| SSH tunnel | host and user keys | A dropped connection stops signing without a clear error |
| Cloudflare Tunnel and Access | service token | No inbound port on either host |

The procedure below uses Cloudflare. The requirement is the same for all
options: **the transport must reject an unauthenticated client.**

### 1. Publish the hostname

Add a public hostname of type TCP to your tunnel:

```
signer.example.com -> tcp://node:26659
```

Address the node by name. The tunnel daemon resolves the name on the node host.
The address of the node can then change without configuration changes.

### 2. Attach an Access policy

A public hostname is public. `cloudflared access tcp` is a standard client. Any
client can connect to the hostname. Until you attach a policy, the hostname
gives public access to the privval socket.

Both ends are machines. Use a service token. Do not use an identity provider
login.

1. Open Zero Trust → Access → **Service Auth**. Create a service token. The
   Client Secret is displayed once. You cannot retrieve it later. Use a
   non-expiring token. An expiry date is a date when the validator stops
   signing.
2. Open Zero Trust → Access → **Applications**. Add a **Self-hosted**
   application for `signer.example.com`.
3. Add one policy. Set the action to **Service Auth**. Include the service
   token.

Do not use the **Allow** action. It requires an identity provider login. A
headless signer cannot complete that login.

To rotate the token, use this order:

1. Create the new token.
2. Add it to the policy.
3. Restart the client on the signer host.
4. Revoke the old token.

Revoking first disconnects the signer. The validator then produces no blocks.

### 3. Start the client and the signer

```bash
export CF_ACCESS_CLIENT_ID=...access
export CF_ACCESS_CLIENT_SECRET=...

cloudflared access tcp --hostname signer.example.com --url localhost:26659
tmkms start -c tmkms.toml
```

cloudflared reads both values from the environment. Do not pass them as
command-line flags. Flags expose the secret in `ps` output and in shell history.

Start the tunnel first. `cloudflared access tcp` binds `localhost:26659` and
forwards the connection. tmkms then dials the same address that it uses for a
local node, so `tmkms.toml` does not change. If you start tmkms first, it dials
a closed port.

### 4. Verify the policy

Run this command on a host that has no token. The connection must fail:

```bash
cloudflared access tcp --hostname signer.example.com --url localhost:26699
```

Do this test. An application that is not attached to the hostname behaves the
same as a correct one when you test it with a token.

---

## tmkms configuration

Chain-specific values:

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

Three common errors:

- `protocol_version` must be `v0.38` for CometBFT 0.38 and Cosmos SDK v0.53.
  Version `v0.34` is deprecated. It becomes an error in a later tmkms release.
- The state file requires `round` as a string. It also requires an explicit
  `block_id` key. If either is missing, tmkms does not start.
- Incorrect bech32 prefixes do not stop signing. The signature covers bytes. But
  all log output and all printed keys then show addresses for a different chain.

Normal output is one line per vote:

```
signed Proposal:8E3026E331 at h/r/s 6/0/0 (0 ms)
signed Prevote:8E3026E331  at h/r/s 6/0/1 (0 ms)
signed Precommit:8E3026E331 at h/r/s 6/0/2 (0 ms)
```

`Connection refused` before the node starts is normal. tmkms retries.

---

## Migration of a running validator

Do this before other accounts delegate to you. This procedure has a risk of
double-signing. The risk exists while both the node and the signer can sign.

1. Stop the validator. Confirm that it produces no blocks.
2. Copy `priv_validator_key.json` to the signer host. Import it into tmkms.
3. Set the tmkms state from `priv_validator_state.json`. Without this step,
   tmkms starts with no record of signed heights.
4. Start tmkms. Then start the node with `priv_validator_laddr` set.
5. Confirm that the node signs blocks again.
6. Delete the key from the rented host. If you skip this step, the key stays on
   the host that you do not control.
