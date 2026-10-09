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
refresh, your handle's renewal and a suggested move after a switch, appear as
reminders.

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

## Identity

The **Identity** screen shows your registration. One recovery phrase holds a
series of identities, and the chain accepts each only once, so every
registration after your first uses the wallet's next identity. See
[Your identities](./registering.md#your-identities).

- **Renew registration.** A registration lasts a year. Once it has ended, tap
  Renew registration and scan your passport: the wallet registers it to its
  next identity, from the same recovery phrase. No new wallet or phrase.
- **Switch identity.** While your registration is live, you can move it to
  another wallet on the phone, or to **a fresh identity in this wallet**. The
  fresh identity helps only if this identity's secret leaked by itself,
  outside the phone (for example in a proof's witness file or a log). If the
  phone may be compromised or your recovery phrase may have leaked, create a
  new wallet instead, with a new phrase, and switch to it. If your handle's or
  caretaker vote's lease ends within 30 days, the screen asks you to renew it
  first (**Renew first**). A passport can switch once a day (UTC).
- **Bring your handle and caretaker vote.** After a switch or renewal, the
  identity you left still holds them. Identity offers to move each to the new
  identity, as a private transaction proven with both identities' secrets.
  Within one wallet the one phrase is enough and the fee comes from its
  private ERTH. The app suggests a random time to move, 6 hours to 3 days
  after the switch or renewal, so the move is not linked to it by timing; it
  reminds you then and never moves on its own. Move **by** the date the app
  shows: the end of the handle's or vote's lease, after which neither can move
  and the old identity cannot renew it. The suggestion is always at least 3
  days before that date, or the app says **Move now**. Move before switching
  again, too: a move goes only from an identity to the one that replaced it.
- **Before your registration's year ends**, renew your handle and refresh your
  caretaker vote, so that their leases outlast the registration. The app
  reminds you from 30 days before the end.

See [Switching identity](./registering.md#switching-identity) and
[Moving your handle and vote](./registering.md#moving-your-handle-and-vote).

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
anyone else. To hand stake over, unstake it and send the ERTH. Staked ERTH
also lets you vote in governance and in the Groundworks fund.

New stake is bonded in the block your transaction lands in and earns from
then on. Unstaking takes effect at the end of the day's **epoch**, together
with everyone else's. The amount and the validator are public, you are not.

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
- **Groundworks**: your stake votes. Pick a split once; the app sends one
  transaction per validator you stake with, and from then on every stake
  transaction carries your split, so the vote follows your stake. Nothing is
  locked: your stake keeps earning and moves freely. Each vote is public (its
  validator, size and split); only your phone can tie it to you, but while you
  vote your stake transactions at a validator are linked to one another. A
  vote lasts one year from when it was cast or last carried forward; the app
  reminds you before it expires, and voting again renews it. **Stop voting**
  removes your votes. An expired vote stops counting until you vote again. If
  governance resets the Groundworks votes, your split counts for nothing until
  you vote again. Stake moved in by a redelegation starts voting only after
  its label clears (about 21 days) and a later stake transaction there
  carries it.

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
