---
sidebar_position: 3
title: The two fund streams
---

# x/allocation: the two fund streams

Updated 2026-09-24, as of v0.9.1. `x/allocation` runs both voted funds on one
engine: the Caretaker stream (one vote per human) and the Groundworks stream
(bonded stake). It replaced two near-duplicate engines in `x/personhood` and the
old `x/deflation`. Every fix had been made twice before the merge.

## Structure

Both streams live in one module, keyed by stream id:

```go
STREAM_ID_CARETAKER   = 1 // one vote per human, 1 ERTH/sec
STREAM_ID_GROUNDWORKS = 2 // stake-weighted,      1 ERTH/sec
```

Every piece of state carries the stream: `Options[(stream, id)]`,
`Voters[(stream, addr)]`, `RewardIndex[stream]`, `TotalWeight[stream]`,
`Epoch[stream]`, `OptionSeq[stream]`, `IntegratedOptions[(stream, id)]`. Option
ids restart per stream, so Caretaker #1 (registration rewards) and Groundworks #1
(LP rewards) are different options.

The streams differ only in where weight comes from:

```go
type WeightSource interface {
    Weight(ctx context.Context, addr []byte) (math.Int, error)
}
```

`x/personhood` registers the Caretaker source. A live, unrevoked registration is
`HumanVoterWeight`; since v0.9.1 a registration under a revoked Document Signer
weighs zero at once. The Groundworks source is internal (`GetDelegatorBonded`),
kept current by staking hooks.

## Option kinds

- **INTEGRATED** options settle every block into a registered handler:
  `lp_rewards` (x/dex) and `registration_rewards` (x/personhood). Everything on
  earth-1 today is integrated.
- **ADDRESS** options accrue to a recipient who claims. On Caretaker anyone can
  add one for a fee. On Groundworks only governance can.

## Who can change the slate

| Action | Caretaker | Groundworks |
| --- | --- | --- |
| Add an option | Anyone, for a fee | Governance |
| Remove an option | Prune sweep only | The assembly alone, by removal ballot (`RegisterChamber`, `RemoveGroundworksOption`) |
| Reset every vote | Nobody. `MsgResetAllocations` rejects it (v0.8.0) | Governance |

A captured Caretaker slate has no on-chain clear. Recovery is a binary upgrade.
That was a deliberate trade: a reset is a mute button on the persons axis, and
stake should not hold one.

## Money: solvency rules

Emission is minted into the module when a stream's index advances, never on
claim. So the module's ERTH balance must always cover what options are owed.

- `SummedAccrued` is the running sum of every option's `Accumulated`, maintained
  on every write through `setOption`. Anything that removes an option (the prune
  burn, removal) must decrement it (v0.7.0 fix).
- `Residue` is minted emission no option can reach, swept to the community pool.
  It is carried through genesis, and `SummedAccrued` is rebuilt on import
  (v0.7.0).
- `CheckSolvency` runs in EndBlock through `AssertHotInvariants`. Short means a
  chain halt, by design.
- **The options' share of each interval is rounded UP** (`ceilQuo`, v0.9.1).
  Rounded down, a lazily settled option collected the fractions of every block it
  skipped, the module went short, and EndBlock halted. On earth-1 residue is now
  about zero, and per-option truncation dust sits on the account as surplus,
  bounded by 1 uerth a block plus 1 a settle.

## Dependency direction

`x/allocation` depends only on staking and bank. `x/dex`, `x/personhood` and
`x/assembly` call into it through registries populated at wiring time
(`RegisterWeightSource`, `RegisterIntegratedHandler`, `RegisterChamber`,
`RegisterResidueSink`). `x/personhood` reaches back only through a narrow
interface it declares itself (`AdvanceIndex`, `ClearVoter`, `DrawFromOption`). A
two-way keeper dependency would deadlock depinject.

## Invariants worth knowing

- The two streams' epochs are independent (`TestResetAllocationsIsPerStream`).
- `MaxVoterOptions` is a DoS bound. The personhood expiry sweep unwinds a lapsed
  human's split option by option in BeginBlock, and no one pays gas for it.
- Caretaker weight is flat. Bonded stake must never enter the shared engine.
- `x/personhood` runs before `x/allocation` in BeginBlock, so a lapsed human's
  weight is returned before the block's emission settles.

## Known open items

From the 2026-09-23 review (see the [security review](./security-review.md)):

- A staking hook re-applies a voter's old split after a Groundworks reset,
  without checking the epoch.
- The assembly's removal runs in EndBlock without a cache context, so an error
  there halts the chain.

## Verifying a change

Run a chain, not just the tests. Three separate reorganisation bugs passed every
test and would have broken a live chain: missing module-account permissions, a
renamed genesis key, and a malformed proto. At minimum, check at a fixed height:

- the bonded pool equals the sum of validator tokens;
- `personhood params` shows 7 verifying keys;
- both streams accrue and allow a claim;
- `AssertInvariants` passes.
