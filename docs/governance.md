---
sidebar_position: 5
---

# Governance

A vote controls every change to the rules. There is no admin key. The authority
for each privileged action is a module account **with no private key**. Only a
passed proposal can make that account act.

## Scope of governance

- The passport certificates that the chain accepts, and the revocation of a
  compromised certificate.
- The proof verifying keys.
- The parameters of each module: fees, unbonding times, and emission
  destinations.
- The start of the liquidity auction.

Every one of these needs both houses below.

## Two houses

A proposal needs **both** of two separate votes.

| House | Who votes | Weight | To pass |
| --- | --- | --- | --- |
| Stake | validators, and anyone holding staked ERTH | the amount bonded | see the table below |
| Assembly | anyone with a live registration | one private vote each | two thirds of the votes cast, or three quarters on the expedited track |

The assembly votes on **every** proposal. There is no proposal that stake can
pass on its own, upgrades of the software included.

**A proposal that no human votes on fails.** There is no quorum in the assembly
and no minimum turnout, so silence is a refusal rather than consent. This is
deliberate, and the cost is real: if nobody votes, nothing changes, including a
fix for a problem everyone agrees about. Vote.

The assembly can only agree or refuse. It cannot write a proposal, spend money,
or set a parameter. One thing is an exception: registered humans can vote to
**remove an option from the Groundworks fund**, and that vote needs no stake
vote at all.

There is no abstain in the assembly. Two thirds is measured against the votes
cast, so an abstention and a missing vote are the same thing.

## Assembly votes are private

An assembly vote is a proof that the voter is one of the registered humans,
with a tag that is the same every time that person votes on that ballot and
unrelated to anything else they do. So the chain can count one vote per
person, and let a person change their vote, without knowing who anyone is. The
running tally is public. Who voted for what is not. See
[Privacy](./privacy.md#how-claims-and-votes-stay-unlinkable).

A first-time registrant can vote at once, on every ballot that is open,
including ones that opened before they registered. The bar falls only on an
identity that replaced another: a **switch** (the same passport registered from
a new wallet) or a **re-entry** (the same passport registering again after its
registration lapsed). Such an identity cannot vote on a ballot that opened
before the switch or re-entry, or less than a day after it. Its predecessor may
already have voted there, and the two votes would carry unrelated tags, so this
is what stops one passport voting twice under two secrets. It votes normally on
every ballot that opens later.

## Stake votes

Most staked ERTH is held privately, as derth notes in `x/shieldedstaking`'s
own note tree (see [Staking](./using-the-app.md#stake)). The app proves that
your notes at a validator existed, unspent, when the proposal opened, and casts **one vote per
validator** you stake with. Nothing is spent, so the same stake can vote on
every open proposal. The vote's weight, your notes there rounded down to three
significant figures, and its validator are public. The voter is hidden. A stake
vote is final: each note publishes a tag for the proposal, and a tag can be
used once.

The snapshot decides. A note votes if it existed, unspent, when the proposal
opened. Stake you spend or move after that (unstake, top up, redelegate) still
votes on that proposal with the note it was in, so you lose nothing by moving.
Only stake **added** after the snapshot cannot vote on it: the new note was not
in the snapshot. So no unit of stake votes twice.

Validators vote with their own self-bond in the open. A validator's vote also
covers the stake delegated to it that did not vote, as on other Cosmos chains.
If you disagree with your validator, vote yourself and your weight is taken
out of theirs.

Groundworks positions vote too, proven by the position's owner tag, with their
weight public like the position. A position's vote can be replaced.

A slash during the vote shrinks the votes behind that validator with it, so
the votes counted never exceed the stake that is actually bonded.

## Fixed rules

The rules of the assembly are fixed in the software. There is no parameter for
its threshold, because a threshold that governance could set is one governance
could set out of reach.

## Vote parameters

| | Deposit | Voting period | Yes votes required |
| --- | --- | --- | --- |
| Normal | 1 ERTH | 7 days | Two thirds |
| Expedited | 5 ERTH | 1 day | Three quarters |

Both tracks require a **33.4% quorum**. A **33.4% veto** defeats a proposal.
Bonded stake determines voting power. These figures describe the stake house
only; the assembly has its own threshold and no quorum, as above.

The threshold is two thirds and not a simple majority. A rule that changes at
50.1% changes at each close vote. Users cannot build on parameters that change
frequently.

The expedited track requires more agreement, not less. It reduces deliberation
from seven days to one day. The higher threshold compensates for the shorter
period. This applies in both houses: the assembly also asks three quarters of an
expedited proposal.

An expedited proposal that falls short in either house is **not rejected**. It
becomes an ordinary proposal and is decided under ordinary rules. Its voting
period is the ordinary seven days counted from when voting **first** opened, so
about six days remain, not seven more. The deposit stays with it.

- **Stake votes carry over.** The stake house tallies again at the new end with
  the votes already cast. A private stake vote stays final: it cannot be cast
  again on the same proposal. Stake that has not voted can still vote.
- **Human votes start again.** The assembly opens a new ballot for the ordinary
  round, counted from zero, and everyone votes afresh, including those who
  voted in the expedited round.

Submit the proposal with the full deposit. A proposal with a partial deposit
stays in the deposit period and the voting period does not start.

## Revoking a compromised certificate

Governments sign passports with **Document Signer** certificates. If one is
stolen, or a government misuses it, it can sign passports for people who do not
exist. The chain's answer is to revoke that certificate, which takes a governance
proposal. Use the **expedited** track: one day instead of seven.

When the revocation passes:

- No new registration signed by that certificate is accepted.
- Registrations already made with it are removed in batches over the following
  blocks. Once the identity roots from before the removal age out, a removed
  registration can no longer claim ANML, vote in the assembly, or cast or
  refresh a Caretaker split.
- Votes those registrations already cast stay where they are, because the
  chain cannot tell which votes were theirs. A Caretaker split lapses within a
  year without a refresh, which a removed registration cannot make.

**The registrations under a certificate cannot vote on revoking it.** Each
registration commits to its Document Signer and its country, and the vote's
proof shows the voter's are not the ones being revoked, without showing what
they are. So a forger with enough fake registrations cannot vote down its own
revocation. Everyone else votes as normal.

- A proposal that revokes **one** Document Signer excludes that signer's
  registrations only.
- A proposal that revokes several signers, or a Country Signing CA, excludes
  **every registration from that country** for that vote. All its revocations
  must belong to one country.
- A proposal that spans two countries cannot be voted on and fails. Submit one
  proposal per country.
- A revocation must stand alone, at the top level. A proposal that carries a
  revocation together with **any other message**, or wraps a revocation inside
  another message such as an authz `MsgExec`, cannot be voted on in the
  assembly and fails. Otherwise a change the excluded registrations had no say
  in could ride along with the revocation.

This is blunt. Real people whose passports that certificate signed lose their
registration too, and have to register again with a passport signed by a
different certificate. Each registration names its certificate and country
publicly, so who is affected can be counted before the vote.

Plan for turnout. An expedited revocation needs three quarters of the human votes
cast within one day. If it falls short it is not lost, but it then runs to the
end of the ordinary seven days from its start, about six more, and the humans
must vote again. Tell people a revocation vote is open.

The [trust store runbook](./operations/trust-store-runbook.md) covers the
procedure. Write it down before an emergency, not during one.

Revoking a **Country Signing CA**, the root a country's Document Signers chain
to, stops new registrations under it but does not remove existing ones. Those are
removed by revoking the individual Document Signers.

## Current state of the chain

Earth starts with **one validator**. One party therefore controls the stake
house. This is not the result of a special key. It is the result of being the
only staker. That condition ends when other parties stake or run validators;
there is no allocation list that prevents them from doing so.

The assembly is small at the start for the same kind of reason: it holds as many
votes as there are registered humans. With few registrations, few people decide —
and because two thirds is measured against the votes cast, a single voter is one
of one and carries a proposal alone. The check on the stake house is real from
the first day, but it is only as broad as the register behind it.
