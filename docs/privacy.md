---
sidebar_position: 2.5
---

# Privacy

On Earth, what you own and what you do as a person is private. What you do to
the shared economy is public. This page lists which is which, how the private
parts work in plain terms, and what can still leak.

The short version:

| Private | Public |
| --- | --- |
| Your ERTH and ANML balances, and every send, swap and stake | That a transaction happened, and its fee |
| Which wallet belongs to which registered human | That a passport registered, its country and its signing certificate |
| Your ANML claims | That someone claimed |
| Your assembly votes | The running tally |
| Who cast each Caretaker split | The split itself, and how much each option earns |
| Who owns each Groundworks position | The position: its size, its validator and its split |
| Who staked, and how much each person holds | The amount entering or leaving each validator |

Allocations are public acts. Pointing a share of the chain's issuance at
someone is a decision about money that belongs to everyone, so the decision is
visible. The person making it is not.

## What never leaves your phone

- Your name, date of birth, passport number and photo. The app reads them from
  the chip to build a proof and then discards them.
- Your secrets. Everything private on Earth is derived from your recovery
  phrase: the key that owns your notes, the secret behind your registration, and
  one-time keys for Groundworks positions. The recovery phrase is still the only
  backup.

## Shielded and transparent ERTH

ERTH exists in two forms.

- **Shielded ERTH** sits in the shielded pool as **notes**. A note is a sealed
  record that says "this much of this token belongs to whoever holds this key".
  The chain stores only a fingerprint of each note, so it can check that a note
  exists and has not been spent without learning its owner or its amount. When
  you send shielded ERTH, the chain sees that some notes were spent and some new
  ones created, and a proof that the amounts balance. It does not see who, to
  whom, or how much.
- **Transparent ERTH** is ordinary ERTH in an ordinary account, visible like on
  any other chain. It exists for the public parts of the economy: validators'
  self-bond, liquidity pools, IBC transfers, exchanges, contracts, and payouts
  to the addresses of fund options.

You move between the two by **shielding** (transparent to shielded, which shows
the amount going in and the account it came from) and **unshielding** (the
reverse, which shows the amount coming out and the account it goes to). Inside
the pool, nothing is linked.

The app keeps your ERTH shielded by default. Unshield only what you need to
make public.

## ANML is always shielded

ANML exists only as notes. It cannot sit in an ordinary account, so it cannot be
sent to an exchange address or a contract. It is claimed into a note, sold from
a note, and provided as liquidity from a note. Its only public appearances are
in the ANML/ERTH pool's reserves and in the buyback, which burns it.

## Registration has no wallet

Registering proves you hold a genuine passport, as before. What changed is what
the proof is tied to. It is no longer tied to a wallet address. It is tied to a
**secret commitment**, a value derived from a secret on your phone that the
chain can check later proofs against without ever seeing the secret.

The chain records each registration under its passport nullifier, the one-way
tag that stops one passport registering twice. That record holds the
registration time, the passport's country and its Document Signer certificate.
It holds no address. Your registration reward and your first ANML are paid into
notes that only your phone can find.

From then on, everything you do as a person is a fresh proof: "I am one of the
registered humans, and here is a tag that stops me doing this twice". The proof
does not say which human.

### Switching phones or secrets

Registering again with the same passport while your registration is live is a
**switch**. The old entry is retired, a new one takes its place, and nothing is
paid twice. A switch resets the waiting periods below, because the chain cannot
tell the new entry and the old one apart from anyone else.

## How claims and votes stay unlinkable

Each private action carries a **nullifier**: a tag computed from your secret
and the action's scope, such as "the ANML claim for 3 October" or "ballot 12".
The same person always produces the same tag for the same scope, so a second
claim that day or a second vote on that ballot is refused, or replaces the
first. Different scopes produce unrelated tags, so nobody can tell that
Monday's claim and Tuesday's claim, or a claim and a vote, came from one person.

The tags come from your secret, never from your passport. The government that
issued your passport knows your passport number, so a tag built from it could
be recomputed by them. A tag built from a secret on your phone cannot.

Because the chain cannot follow a person across actions, a new or switched
registration waits before some actions open:

| Action | How often | Opens after you register |
| --- | --- | --- |
| Claim 1 ANML | once a UTC day | the day after tomorrow (registering pays your first ANML at once) |
| Vote in the assembly | once a ballot, replaceable | ballots that open after you register |
| Caretaker split | refreshed every 30 days by the app | 30 days |
| Referrer binding | refreshed every 30 days by the app | 30 days |

The waits stop one passport from acting twice under two secrets. A Caretaker
split counts for 30 days and then lapses unless refreshed, because the chain
cannot find a lapsed person's split to remove it. The app refreshes it for you.

## Referrals are public

A referral pays an address, so it is a public act. A registered human can
**bind** an address as their referral address with a membership proof. The
chain then knows that some live human vouches for that address, not which one.
A registration that names the address pays the referrer's half of the reward
to it in transparent ERTH. The registrant's half is still a private note.

## Private staking

Staking is private too. The shielded pool is the only delegator to validators.
You hold its delegation as notes.

- **Delegation tokens.** Delegating to a validator turns shielded ERTH into a
  delegation token for that validator, also held as a note. Each validator's
  token has an **exchange rate** into ERTH. Rewards are restaked for everyone
  at once, so the rate rises. You never claim rewards: your tokens become worth
  more ERTH.
- **Daily epochs.** Delegations and undelegations take effect together once a
  day, at the end of an epoch. That batches everyone's changes, which hides
  when each one was made.
- **Unbonding** still takes 21 days, from the epoch it is processed in. You hold
  an unbonding note meanwhile, and redeem it for ERTH when it matures. If the
  validator is slashed during unbonding, the note pays out less, exactly as an
  ordinary unbonding would.
- **Slashing** lowers the validator's exchange rate, so every holder of its
  token shares the loss in proportion.

What is visible: the amount entering or leaving each validator, and each
validator's rate. Not who holds the tokens.

Validators bond their own stake publicly from their operator address. Running a
validator is a public role. Ordinary delegation from a transparent account is
not possible on Earth.

### Voting with stake: spend to vote

To vote on a proposal with your stake, the app spends your delegation note
against a snapshot taken when the proposal opened, and immediately gives you a
fresh note of the same value. The vote counts with the note's weight. The spent
note cannot vote again, and the fresh one did not exist at the snapshot, so it
cannot either. Votes are final.

The weight and the validator are public. The voter is not. A validator's vote
also covers any of its stake that did not vote, as on other Cosmos chains.

### Groundworks positions

Voting in the Groundworks fund is a public act weighted by stake, so it uses a
**position**: delegation tokens locked into a public record with a split. The
position shows its size, its validator and its split. Its owner is a one-time
key that only your phone holds. Locked tokens keep earning. Unlock them and they
return to you as a note.

## Fees: paid in ERTH, half burned

Every transaction pays a fee in ERTH. There are no free transactions on chain.

Private transactions are not signed by any account. They pay their fee from a
shielded ERTH note, with a proof that the fee was taken from a note you own. An
action that produces ERTH, such as selling ANML or redeeming an unbonding note,
can pay its fee from what it produces. Someone who holds only ANML can sell it
and pay from the proceeds.

Half of every fee is burned and half goes to validators, as before.

Your first fee is covered by Earth's backend. Before you register, the app asks
the backend for gas. The backend checks the registration you are about to send
is valid, and shields a little ERTH into a new note for you. It learns that a
passport is about to register, which the chain shows publicly anyway, and
nothing about where the note goes next. After that, your registration reward
pays your fees.

## What can still leak

Privacy here is strong, not perfect. Know the limits.

- **A small crowd hides you less.** Your actions hide among everyone else's.
  At launch there are few registered humans and few notes, so a claim or a
  vote is one of only a handful. The privacy grows with the network. The app
  jitters its automatic actions and staking batches at epoch ends to help.
- **Timing.** If you shield 100 ERTH at noon and someone unshields 100 ERTH at
  12:01, an observer can guess. Wait between moving in and out, and avoid
  round, distinctive amounts.
- **Reusing a transparent address.** Everything a transparent ERTH address does
  is public and linked together. If you shield from and unshield to the same
  address, or post it with your name, it is not private. Use a fresh address
  for each public purpose.
- **Amounts at the edges.** Shielding, unshielding, delegating, swapping from a
  note and locking a position each reveal their amount and, where it applies,
  the pool or validator. Only the owner is hidden.
- **What the registration shows.** A registration still shows the passport's
  country and its Document Signer certificate, which narrow down roughly which
  office issued it and when. With few registrants from one country, that is a
  small group.
- **Your network connection.** Whoever relays your transaction sees your IP
  address. Use a VPN or Tor if that matters to you.

## The indexer, and why your wallet downloads everything

To find your notes, your phone has to look at every note. Notes carry an
encrypted message that only the owner's key can open, so the phone downloads
all of them and tries to open each one. It also downloads every spent tag and
every registration entry, and rebuilds the chain's trees itself.

It never asks "which notes are mine?" or "where is my registration?". Asking
would tell whoever answered. Downloading everything tells them nothing.

Earth runs an **indexer** that packages this data compactly so phones can sync
quickly. It serves only full ranges, the same bytes to everyone, and keeps no
accounts. Anyone can run their own from a full-history node, and the app checks
what it downloads against the roots the chain publishes, so a dishonest indexer
can withhold data but cannot forge it.

## What Earth's backend sees

The backend does two things: it pays gas for a first registration and it runs
the indexer.

- **Gas.** It sees the registration you are about to send, which the chain is
  about to publish anyway, and pays a note it cannot follow. It keeps the
  passport nullifier and the month, so one passport is funded once a month.
- **Indexer.** It sees which ranges your phone downloads, like any website sees
  page requests. Every phone downloads the same ranges.

It never sees passport data. The app contains no ads and no advertising or
tracking SDKs.
