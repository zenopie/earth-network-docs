---
sidebar_position: 2.5
title: Use your own node
---

# Use your own node with Earth Wallet

By default Earth Wallet sends your transactions to, and reads the chain from,
Earth's node at `lcd.erth.network` and `rpc.erth.network`, behind Cloudflare.
That node receives your IP address, every transaction you broadcast and the
address the app asks about. Earth runs it under a no-logs policy, but that is a
promise you have to trust (see
[What Earth's servers see](../privacy.md#what-earths-servers-see)). Pointing
the wallet at a node you run removes that trust.

## What moves to your node, and what does not

Your node takes over everything the wallet asks the chain directly: balances,
staking and liquidity reads, transaction history, and broadcasting every
transaction.

These still come from Earth's backend (`api.erth.network`), because they need
data a plain node does not serve:

- the **private note streams** (the indexer your phone syncs notes from),
- the **handle directory**,
- the **gas grant** for a first registration or a switch,
- **circuit downloads**.

The note streams, the directory and the circuits are the same bytes for every
phone, and the app checks the notes against the roots the chain publishes. The
gas grant is the exception: it carries your registration's passport nullifier.
Ask for it over a VPN or Tor if that matters to you.

## 1. Run a node

Follow [Join the network](./join.md). A state-synced node is enough for
balances and broadcasting. Transaction history comes from the node's
transaction index, so keep `[tx_index] indexer = "kv"` (the default) in
`config.toml`; a state-synced node has history only from the height it synced
at.

## 2. Turn on the REST API and RPC

`app.toml`:

```toml
[api]
enable = true
address = "tcp://127.0.0.1:1317"
```

`config.toml` (the default):

```toml
[rpc]
laddr = "tcp://127.0.0.1:26657"
```

Both listen on localhost only. The next step puts HTTPS in front of them.

## 3. Serve them over HTTPS

Phones refuse plain HTTP to a remote host, so the node needs a domain name and a
certificate. A minimal [Caddy](https://caddyserver.com) configuration, which
obtains the certificates itself:

```text
lcd.example.org {
    reverse_proxy 127.0.0.1:1317
}
rpc.example.org {
    reverse_proxy 127.0.0.1:26657
}
```

Open port 443. Do not put a CDN in front of it: the point is that nobody but
you sees this traffic.

Anyone who learns the two addresses can use your node too. If you would rather
not expose it to the internet, reach it over a VPN you run and use the VPN address in the wallet. The app
accepts plain HTTP only to a local address: 10.x, 172.16–31.x, 192.168.x,
link-local, IPv6 fc00::/7, a `.local` name, or `localhost`, so a WireGuard address in those ranges works
without a certificate. Tailscale's 100.x addresses count as remote and need
HTTPS (Tailscale can issue the certificate). Anyone on that network can read
plain HTTP, so use it only on a network you control.

## 4. Point Earth Wallet at it

In Earth Wallet, open **Settings → Network** and enter your node's REST (LCD)
and RPC addresses, for example `https://lcd.example.org` and
`https://rpc.example.org`. Both are required. The wallet then sends its chain
traffic only there.

Before saving, the wallet checks that the node follows the live chain:

- **Same genesis.** It reads the node's genesis from the RPC
  (`/genesis_chunked`) and compares it with the live chain's. Every node keeps
  its genesis, so a state-synced node passes. A node left over from an earlier
  earth-1 launch, or another network using the same name, is refused.
- **Same node.** The LCD and the RPC must report the same block at a recent
  height, so they belong to one node.
- **Synced.** A node whose latest block is more than 10 minutes old is refused
  until it catches up.

If your node later falls behind or stops, the wallet shows stale balances or
cannot send until it is back.

The web app at erth.network always uses Earth's node.
