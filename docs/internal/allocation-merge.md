---
sidebar_position: 3
title: Allocation merge
---

# Merge of the two allocation streams

This is step 3 of the module reorganisation. It is **complete**. Step 1 moved
tokenomics into `x/earth`. Step 2 renamed `x/caretaker` to `x/personhood`.

`x/personhood/keeper/allocation.go` and `x/deflation/keeper/allocation.go` were
near-duplicates of one engine, maintained in two places. Each fix so far —
epoch-reset semantics, the voter split cap, the zero-percent rejection — was
applied two times. They are now one module.

## Current structure

`x/allocation` owns both emission streams. A stream id identifies them:

```go
STREAM_ID_CARETAKER   = 1 // one vote for each human, 1 ERTH/sec
STREAM_ID_GROUNDWORKS = 2 // stake-weighted,          1 ERTH/sec
```

Each item of state carries a stream prefix: `Options[(stream, id)]`,
`Voters[(stream, addr)]`, `RewardIndex[stream]`, `TotalWeight[stream]`,
`Epoch[stream]`, `OptionSeq[stream]`, `IntegratedOptions[(stream, id)]`. Option
ids restart for each stream.

The streams differ in one place only, behind one interface:

```go
type WeightSource interface {
    // Weight returns the voter's current weight, or zero if they may not vote.
    Weight(ctx context.Context, addr []byte) (math.Int, error)
}
```

`x/personhood` registers the caretaker source: a live registration maps to
`HumanVoterWeight`. The groundworks source is internal to `x/allocation`
(`GetDelegatorBonded`), because that module already holds the staking hooks that
keep the weight current.

`x/deflation` is removed. `x/dex` registers its `lp_rewards` integrated handler.
`x/personhood` registers `registration_rewards`.

## Direction of dependencies

The registries `RegisterWeightSource` and `RegisterIntegratedHandler` are maps
on the keeper. The wiring of other modules populates them. This keeps the
dependency graph a tree. `x/allocation` depends on staking and bank only. `x/dex`
and `x/personhood` call *into* it. A keeper-to-keeper dependency in both
directions would deadlock depinject.

`x/personhood` calls back through a narrow interface that it declares itself:
`AdvanceIndex`, `ClearVoter`, `DrawFromOption`. It does not import the
allocation keeper.

## Invariants

- **The two epochs are independent.** A governance reset of one stream must not
  affect the other. `TestResetAllocationsIsPerStream` verifies this.
- **`MaxVoterOptions` is a DoS bound, not a usability rule.** The expiry sweep in
  `x/personhood` clears the vote of a lapsed human from BeginBlock. It unwinds
  the split one option at a time, and no account pays gas for this. The sweep
  limit multiplies this work within one block.
- **The weight of the caretaker stream is flat.** Bonded stake belongs to the
  groundworks weight source and to the staking hooks. It must never enter the
  shared engine.
- **Option ids are per stream.** `RegistrationRewardOptionID` and
  `LPRewardsOptionID` are both 1. Under one module, global ids would collide.
- **`x/personhood` runs before `x/allocation` in BeginBlock.** The sweep must
  return the weight of a lapsed human before the stream settles the emission of
  that block.

## Verification

Run the chain. Do not rely on the test suite alone. Three separate defects
during this reorganisation passed each test and would still have broken a
running chain: missing module account permissions, a renamed genesis key, and a
malformed proto file.

Minimum checks: the bonded pool balance equals the sum of validator tokens at a
fixed height; `personhood params` returns 7 verifying keys; both streams accrue
and permit a claim.
