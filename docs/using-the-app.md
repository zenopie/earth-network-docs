---
sidebar_position: 4
---

# Using the app

The app keeps your ERTH and ANML **shielded**: private notes that only your
phone can find. It shows one balance per token. Every action below pays a small
fee in ERTH from that balance. See [Privacy](./privacy.md) for what each action
reveals.

The app syncs by downloading every note on the chain and keeping yours. The
first sync after installing or restoring takes longer than the ones after it.

**The app does nothing on its own.** Every transaction is one you confirmed.
Things that need doing regularly, such as the daily ANML claim, the Caretaker
refresh and your handle's renewal, appear as reminders.

## Send and receive

Send to a **shielded address** or a **handle**, and the transfer is private:
nobody sees the sender, the receiver or the amount.

To pay a handle, type `@alice` (or just `alice`). The app downloads the whole
handle directory, checks Alice's entry against the chain, and shows the handle
with the start and end of her address before you confirm. If the handle is not
live, or the two copies disagree, nothing is paid.

Send to an ordinary **transparent address**, such as an exchange deposit
address, and the ERTH leaves the pool. The amount and the receiving address
are public. Receiving from a transparent address works the other way: the app
shields it, and the amount and the sending address are public.

ANML can only be sent to a shielded address or a handle.

## Your handle

Once registered, you can claim a **handle**: a short name of 3 to 32 lowercase
letters, digits and dashes that people can pay instead of your address. It
lasts a year. Renew it before it expires (the app reminds you in the last
month), or during the 30-day renewal window after it expires, when it no longer
receives payments but is still reserved for you. After that anyone can
claim it. You can change to another free handle, or release yours, at any time.

Your handle is also your referral link: `https://erth.network/ref/<handle>`.
Someone who registers from it names you as their referrer, and the chain pays
your half of their registration reward into a private note for you.

## Swap

Each pool pairs one token with **ERTH**. A swap between two other tokens
therefore uses two hops through ERTH. This keeps liquidity in one place. It does
not divide liquidity between all possible pairs.

The fee is **0.3% for each hop**. The chain charges it in ERTH. Half of the fee
stays in the pool for the liquidity providers. It destroys the other half.

Swaps from your shielded balance are private: the pool and the amounts are
visible, you are not. The transaction fee comes from your shielded ERTH, so
keep a little ERTH to sell ANML.

## Provide liquidity

Deposit both sides of a pool from your shielded balance. You receive LP shares
as a private note. You then earn a part of the fees. If voters direct the
Groundworks fund to LP rewards, you also earn a part of that fund.

The pool and the amounts going in and out are public. Who holds the shares is
not. LP share notes stay in the shielded pool: they cannot be unshielded.

A withdrawal takes **7 days**. Your liquidity continues to work and to earn for
this period. When it ends, both sides are paid to you as notes, with nothing
more to send. The only cost of a withdrawal is the delay.

## Stake

Delegate shielded ERTH to a validator. You receive that validator's delegation
token, **derth**, as one private note per validator: staking more with the same
validator adds to the note you have. Rewards are restaked for everyone once a
day, so each derth is worth more ERTH over time. The app shows the ERTH your
derth is worth. Stake notes are owner-locked: they cannot be sent or given to
anyone else. To hand stake over, unstake it and send the ERTH. Staked ERTH also lets you vote in governance and, through a
position, in the Groundworks fund.

Stake changes take effect at the end of the day's **epoch**, together with
everyone else's. The amount and the validator are public, you are not.

**Unstaking** takes **21 days** from the epoch it is processed in. During this
period you earn nothing. This delay makes an attack on the chain expensive.
When it ends, the chain pays the ERTH into your shielded balance by itself.
There is nothing to claim.

**Moving to another validator** is instant: redelegate, and your stake earns at
the new validator from that block, with no unbonding gap. The moved stake stays
answerable for the old validator's past behaviour for about 21 days: if the old
validator is slashed for something it did before you moved, the moved stake
pays its share. Until then you cannot unstake or move that part again. The rest
of your stake is unaffected.

If you hold more than one note at a validator, for example after using two
devices or after a redelegation, merge them: it costs one fee and moves no
value.

Select a validator by commission, by uptime, and by whether the validator
operates its own infrastructure. Do not select a validator because it is the
largest. Concentrated stake is a risk to all users, including the delegators of
the largest validator. If a validator is jailed, move your stake away: its
derth stops earning, and a slash lowers what it is worth.

## Claim ANML

Each registered human can claim one ANML a day into their shielded balance.
The app reminds you when today's claim is open. Unclaimed days do not
accumulate. Daily claims open the day after tomorrow when you register; the
registration itself pays your first ANML.

## Vote a fund

Open the Caretaker fund or the Groundworks fund. Divide your vote between the
options by percentage. Confirm the vote. You can change it at any time.

- **Caretaker**: one vote per registered human, cast anonymously. A split
  counts for a year. The app reminds you to refresh it in the month before it
  lapses; a lapsed split stops counting.
- **Groundworks**: lock staked derth into a **position** with your split. The
  position is public; only your phone can prove it is yours. Locked derth keeps
  earning. Unlock it to merge it back into your note. If governance resets the
  Groundworks votes, your split counts for nothing until you vote again.

Rewards follow the current vote and accrue continuously. A change of vote does
not reset the rewards that you have already earned.

## Vote on a proposal

Every governance proposal has two votes: the assembly, one private vote per
registered human, and the stake vote. You can vote in both.

- An **assembly** vote can be changed until the ballot closes.
- A **stake** vote is one vote per validator you stake with, weighted by your
  derth there. It spends nothing, so you can vote on every open proposal, but
  each stake vote is final.

See [Governance](./governance.md).
