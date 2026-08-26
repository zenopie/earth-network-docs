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

## Vote parameters

| | Deposit | Voting period | Yes votes required |
| --- | --- | --- | --- |
| Normal | 1 ERTH | 7 days | Two thirds |
| Expedited | 5 ERTH | 1 day | Three quarters |

Both tracks require a **33.4% quorum**. A **33.4% veto** defeats a proposal.
Bonded stake determines voting power.

The threshold is two thirds and not a simple majority. A rule that changes at
50.1% changes at each close vote. Users cannot build on parameters that change
frequently.

The expedited track requires more agreement, not less. It reduces deliberation
from seven days to one day. The higher threshold compensates for the shorter
period.

Submit the proposal with the full deposit. A proposal with a partial deposit
stays in the deposit period and the voting period does not start.

## Revocation of a compromised certificate

Governments sign passports with Document Signer certificates. A compromised
certificate permits forged registrations. Revocation therefore uses the
**expedited** track: one day instead of seven.

Revocation is **not retroactive**. Registrations that used the certificate stay
valid. Handle them separately. Each registration names its certificate publicly,
so you can identify them. A single vote that removed all of them would also
remove valid registrations.

The [procedure](https://github.com/zenopie/earth-network-chain/blob/master/docs/TRUST_STORE_RUNBOOK.md)
exists in advance. Do not write a procedure during an emergency.

## Current state of the chain

Earth starts with **one validator**. One party can therefore pass any proposal.
This is not the result of a special key. It is the result of being the only
staker.

This condition ends when other parties stake. There is no allocation list that
prevents them from doing so.
