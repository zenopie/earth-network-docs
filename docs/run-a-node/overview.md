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

There are two consequences:

- Allocate more processor capacity than a chain of this size usually requires.
- Use **state sync**. Do not replay from genesis. A replay verifies each proof
  that the chain has ever received. Sync cost increases with adoption as well as
  with time.

## How to become a validator

There is no allowlist. Acquire ERTH, self-delegate, and submit
`MsgCreateValidator`.

Genesis contained no allocation for validators. You must earn ERTH: by
registration of a passport, by a bid in the liquidity auction, or on the market.

Do not keep the consensus key on the node. See
[Protecting the consensus key](./remote-signer.md).
