---
sidebar_position: 3
title: The two fund streams
---

# x/allocation: the two fund streams

`x/allocation` runs both voted funds on one engine: the Caretaker stream (one
vote per human) and the Groundworks stream (staked positions, and validator
operators' self-bonds). The only accounts that vote are validator operators,
in Groundworks, with their public self-bond.

## Structure

Both streams live in one module, keyed by stream id:

```go
STREAM_ID_CARETAKER   = 1 // one vote per human, 1 ERTH/sec
STREAM_ID_GROUNDWORKS = 2 // stake-weighted,      1 ERTH/sec
```

Every piece of state carries the stream: options, voters, the reward index,
the total weight, the epoch and the option sequence. Option ids restart per
stream, so Caretaker #1 (registration rewards) and Groundworks #1 (LP rewards)
are different options.

The streams differ only in who the voters are and where their weight comes
from.

- **Caretaker.** A split is cast with a membership proof and filed under the
  caster's caretaker nullifier, at one flat weight per human. `x/personhood`
  holds each split's lease (one year by default) and its sweep clears lapsed
  splits, because the chain cannot tell when the person behind a split stops
  being registered. The wallet reminds its owner to refresh the split before
  it lapses; nothing refreshes it automatically.
- **Groundworks.** Weight has two sources, both registered by
  `x/shieldedstaking` as the stream's weight source.
  - **Positions**: staked derth locked with a split. All of one validator's
    positions form ONE weighted voter, keyed `"gwpos/" || validator address
    bytes`, with an absolute weight per option: the validator's epoch rate
    times the sum of `derth x percent` over its live positions. Lock, update
    and unlock adjust those totals exactly, so the work per epoch grows with
    validators, not with positions.
  - **Operators**: a validator's operator account votes with
    `MsgSetAllocations`, weighted by its self-bond. The weight counts only
    while the validator is **Bonded**. When it leaves the active set (or is
    jailed or tombstoned) the vote stays, at weight zero, and the weight
    returns with no new vote when it is Bonded again. A change of self-bond
    re-weighs the operator at once; the validator bonding, starting to unbond
    or being slashed re-weighs it at that block's EndBlock. `x/shieldedstaking`'s own account carries no weight: its
    delegations are the private stake, already counted through positions.

## Option kinds

- **INTEGRATED** options settle into a registered handler:
  - `lp_rewards` (Groundworks, x/dex): paid every block to liquidity providers.
  - `community_pool` (Groundworks, wired in `app.go`): the emergency fund, paid
    into x/distribution's community pool.
  - `registration_rewards` (Caretaker, x/personhood): resolves nothing per
    block. The option's balance stacks, and each registration draws 1e-4 of
    it (half that with no referrer).
- **ADDRESS** options accrue to a recipient who claims. On Caretaker anyone
  can add one for a fee. On Groundworks only governance can.

## Who can change the slate

| Action | Caretaker | Groundworks |
| --- | --- | --- |
| Add an option | Anyone, for a fee | Governance |
| Remove an option | Prune sweep only | The assembly alone, by removal ballot |
| Reset every vote | Nobody: `MsgResetAllocations` refuses it | Governance |

A captured Caretaker slate has no on-chain clear. That is a deliberate trade:
a reset is a mute button on the persons axis, and stake should not hold one.

After a Groundworks reset, a split cast before it counts as zero until its
owner votes again. Positions record the stream epoch their split was cast in,
so the reset is honoured lazily, without walking every position.

## Money: solvency rules

Emission is minted into the module when a stream's index advances, never on
claim. So the module's ERTH balance must always cover what options are owed.

- `SummedAccrued` is the running sum of every option's accrued balance,
  maintained on every write. Anything that removes an option must decrement
  it.
- `Residue` is minted emission no option can reach, swept to the community
  pool, and carried through genesis.
- `CheckSolvency` runs in EndBlock. Short means a chain halt, by design.
- **The options' share of each interval is rounded up.** Rounded down, a
  lazily settled option would collect the fractions of every block it skipped
  and the module would go short.

## Dependency direction

`x/allocation` takes only auth, bank, staking and `x/earth` (for its burn
counters; `x/earth` imports none of the pillar modules). Everything else
registers into it at wiring time:

| Registry | Registered by |
| --- | --- |
| Groundworks weight source | `x/shieldedstaking` (positions, operators' bonded self-bond) |
| Integrated handler `lp_rewards` | `x/dex` |
| Integrated handler `registration_rewards` | `x/personhood` |
| Integrated handler `community_pool` | `app.go`, with x/distribution |
| Residue sink (to the community pool) | `app.go`, with x/distribution |
| Chamber (removes Groundworks options) | `x/assembly` |

`x/personhood` also calls the keeper directly for Caretaker splits and
registration-reward draws. A two-way keeper dependency would deadlock
depinject.

## Invariants worth knowing

- The two streams' epochs are independent.
- `MaxVoterOptions` (20) bounds a split. The personhood sweep unwinds a lapsed
  split option by option in BeginBlock, and no one pays gas for it.
- Caretaker weight is flat. Stake must never enter the Caretaker stream.
- `x/personhood` runs before `x/allocation` in BeginBlock, so a lapsed split's
  weight is returned before the block's emission settles.

## Verifying a change

Run a chain, not just the tests. Reorganisation bugs have passed every test
and still broken a live chain: missing module-account permissions, a renamed
genesis key, a malformed proto. At minimum, check at a fixed height that both
streams accrue and allow a claim, and that the invariants pass.
