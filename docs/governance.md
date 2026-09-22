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
| Stake | anyone with bonded stake | the amount bonded | see the table below |
| Assembly | anyone with a live registration | one vote each | two thirds of the votes cast, or three quarters on the expedited track |

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
becomes an ordinary proposal with a full seven-day voting period ahead of it, and
is voted again under ordinary rules. The deposit stays with it. Votes already
cast do not carry over; the second round is counted from zero.

Submit the proposal with the full deposit. A proposal with a partial deposit
stays in the deposit period and the voting period does not start.

## Revocation of a compromised certificate

Governments sign passports with Document Signer certificates. A compromised
certificate permits forged registrations. Revocation therefore uses the
**expedited** track: one day instead of seven.

Plan for the assembly. An expedited revocation needs three quarters of the human
votes cast within one day. If it falls short it is not lost, but it then takes the
ordinary seven days — so tell people a revocation vote is open rather than
assuming turnout.

Revocation is **not retroactive**. Registrations that used the certificate stay
valid. Handle them separately. Each registration names its certificate publicly,
so you can identify them. A single vote that removed all of them would also
remove valid registrations.

The [procedure](https://github.com/zenopie/earth-network-chain/blob/master/docs/TRUST_STORE_RUNBOOK.md)
exists in advance. Do not write a procedure during an emergency.

## Current state of the chain

Earth starts with **one validator**. One party therefore controls the stake
house. This is not the result of a special key. It is the result of being the
only staker. That condition ends when other parties stake; there is no allocation
list that prevents them from doing so.

The assembly is small at the start for the same kind of reason: it holds as many
votes as there are registered humans. With few registrations, few people decide —
and because two thirds is measured against the votes cast, a single voter is one
of one and carries a proposal alone. The check on the stake house is real from
the first day, but it is only as broad as the register behind it.
