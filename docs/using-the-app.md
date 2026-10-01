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

## Send and receive

Send to a **shielded address** and the transfer is private: nobody sees the
sender, the receiver or the amount.

Send to an ordinary **transparent address**, such as an exchange deposit
address, and the ERTH leaves the pool. The amount and the receiving address
are public. Receiving from a transparent address works the other way: the app
shields it, and the amount and the sending address are public.

ANML can only be sent to a shielded address.

## Swap

Each pool pairs one token with **ERTH**. A swap between two other tokens
therefore uses two hops through ERTH. This keeps liquidity in one place. It does
not divide liquidity between all possible pairs.

The fee is **0.3% for each hop**. The chain charges it in ERTH. Half of the fee
stays in the pool for the liquidity providers. It destroys the other half.

Swaps from your shielded balance are private: the pool and the amounts are
visible, you are not. Selling ANML can pay its transaction fee from the ERTH it
produces, so you can sell ANML with no ERTH at all.

## Provide liquidity

Deposit both sides of a pool. You receive LP shares. You then earn a part of the
fees. If voters direct the Groundworks fund to LP rewards, you also earn a part
of that fund.

Liquidity is a public act: LP shares are held in a transparent account, and the
deposit and withdrawal amounts are public. For the ANML/ERTH pool the ANML side
comes from, and returns to, your shielded balance.

A withdrawal takes **7 days**. Your liquidity continues to work and to earn for
this period. The only cost of a withdrawal is the delay.

## Stake

Delegate shielded ERTH to a validator. You receive that validator's delegation
token as a private note. Rewards are restaked for everyone once a day, so each
token is worth more ERTH over time. The app shows the ERTH your tokens are
worth. Staked ERTH also lets you vote in governance and, through a position,
in the Groundworks fund.

Stake changes take effect at the end of the day's **epoch**, together with
everyone else's. The amount and the validator are public, you are not.

Unbonding takes **21 days** from the epoch it is processed in. During this
period you earn nothing and you cannot transfer the tokens. This delay makes an
attack on the chain expensive. When it ends, the app redeems the unbonding note
for ERTH, paying the fee from it.

To move to another validator, unstake and stake again. There is no instant
redelegation.

Select a validator by commission, by uptime, and by whether the validator
operates its own infrastructure. Do not select a validator because it is the
largest. Concentrated stake is a risk to all users, including the delegators of
the largest validator. If a validator is jailed, move your stake away: its
tokens stop earning, and a slash lowers what they are worth.

## Claim ANML

The app claims one ANML each day for you, at a random time, into your shielded
balance. Unclaimed days do not accumulate. Daily claims open the day after
tomorrow when you register; the registration itself pays your first ANML.

## Vote a fund

Open the Caretaker fund or the Groundworks fund. Divide your vote between the
options by percentage. Confirm the vote. You can change it at any time.

- **Caretaker**: one vote per registered human, cast anonymously. It opens 30
  days after you register, and the app refreshes it every few weeks so it does
  not lapse.
- **Groundworks**: lock staked tokens into a **position** with your split. The
  position is public, its owner is a one-time key on your phone. Locked tokens
  keep earning. Unlock them to return them to your balance.

Rewards follow the current vote and accrue continuously. A change of vote does
not reset the rewards that you have already earned.

## Vote on a proposal

Every governance proposal has two votes: the assembly, one private vote per
registered human, and the stake vote. You can vote in both. An assembly vote can
be changed until the ballot closes. A stake vote is final. See
[Governance](./governance.md).
