---
sidebar_position: 1
---

# Running a node

Any user can run a node. No permission and no stake are required.

- **[Join the network](./join.md)** — binary, genesis file and its checksum,
  peers, gas price, pruning, hardware.
- **[Use your own node](./wallet-node.md)** — point Earth Wallet at a node you
  run, so your transactions and queries do not go through Earth's.
- **[Protecting the consensus key](./remote-signer.md)** — a remote signer for
  validators.
- **[Upgrades](./upgrades.md)** — the procedure for a coordinated upgrade, and
  its failure modes.
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
- The mempool needs no setting. Private transactions have no signer, so
  `earthd` always runs the no-op app mempool and ignores `mempool.max-txs`.

## How to become a validator

There is no allowlist. Acquire ERTH, self-delegate, and submit
`MsgCreateValidator`.

Validators bond their own stake publicly. Nobody else delegates to a validator
directly, and no account can delegate to yours: users stake privately through
the private staking module, `x/shieldedstaking`, which delegates on their
behalf once a day. See
[Join the network](./join.md#how-to-become-a-validator).

Genesis funds one ordinary account: the first validator's operator, with
1,000 ERTH. There is no allocation for other validators. You must earn ERTH: by
registration of a passport, by a bid in the liquidity auction, or on the market.

Do not keep the consensus key on the node. See
[Protecting the consensus key](./remote-signer.md).
