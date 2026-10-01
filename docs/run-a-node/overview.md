---
sidebar_position: 1
---

# Running a node

Any user can run a node. No permission and no stake are required.

The operational guides are stored with the code, so that they stay correct as
the software changes:

- **[Running a node](https://github.com/zenopie/earth-network-chain/blob/master/docs/JOIN.md)**
  — binary, genesis file and its checksum, seeds, gas price, pruning, hardware.
- **[Upgrades](https://github.com/zenopie/earth-network-chain/blob/master/docs/UPGRADES.md)**
  — the procedure for a coordinated upgrade, and its failure modes.
- **[Releases](https://github.com/zenopie/earth-network-chain/releases)** —
  binaries and checksums.

## Differences from other chains

Each registration verifies a zero-knowledge proof **on-chain**. This is unusual
and it uses significant CPU time.

Every private transaction does the same: claims, votes, transfers, swaps and
stake changes each carry a proof.

The consequences:

- Allocate more processor capacity than a chain of this size usually requires,
  on a CPU with the **ADX** instruction set.
- Use **state sync**. Do not replay from genesis. A replay verifies each proof
  that the chain has ever received. Sync cost increases with adoption as well as
  with time. The exception is a node that feeds your own privacy indexer,
  which needs every block (see
  [Join the network](./join.md#4-configure)). The official node behind
  `rpc.erth.network` already keeps full history.
- Keep `max-txs = -1` under `[mempool]` in `app.toml`, the default. Private
  transactions have no signer, and every other mempool setting rejects them.
  See [Join the network](./join.md#4-configure).

## How to become a validator

There is no allowlist. Acquire ERTH, self-delegate, and submit
`MsgCreateValidator`.

Validators bond their own stake publicly. Nobody else delegates to a validator
directly: users stake privately through the shielded pool, which delegates on
their behalf once a day. See
[Join the network](./join.md#how-to-become-a-validator).

Genesis contained no allocation for validators. You must earn ERTH: by
registration of a passport, by a bid in the liquidity auction, or on the market.

Do not keep the consensus key on the node. See
[Protecting the consensus key](./remote-signer.md).
