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
| Your ERTH and ANML balances, and every send, swap, stake and liquidity deposit | That a transaction happened, and its fee |
| Which wallet belongs to which registered human | That a passport registered, its country and its signing certificate |
| Your ANML claims | That someone claimed |
| Your assembly votes | The running tally |
| Who cast each Caretaker split | The split itself, and how much each option earns |
| Who owns each Groundworks position | The position: its size, its validator and its split |
| Who staked, and how much each person holds | The amount entering or leaving each validator |
| Who holds LP shares (deposited from the shielded balance) | Each pool's reserves and total shares, and the amounts going in and out |
| Who pays a handle | The handle directory: each handle and the shielded address it names |

Allocations are public acts. Pointing a share of the chain's issuance at
someone is a decision about money that belongs to everyone, so the decision is
visible. The person making it is not.

## What never leaves your phone

- Your name, date of birth, passport number and photo. The app reads them from
  the chip to build a proof and then discards them.
- Your secrets. Everything private on Earth is derived from your recovery
  phrase: the key that owns your notes, the secret behind your registration,
  and the tags that prove you own a stake position. The recovery phrase is
  still the only backup.

## Shielded and transparent ERTH

ERTH exists in two forms.

- **Shielded ERTH** sits in the shielded pool as **notes**. A note is a sealed
  record that says "this much of this token belongs to whoever holds this key".
  The chain stores only a fingerprint of each note, so it can check that a note
  exists and has not been spent without learning its owner or its amount. When
  you send shielded ERTH, the chain sees that some notes were spent and some new
  ones created, and a proof that the amounts balance. It does not see who, to
  whom, or how much. A transaction spends at most **16 notes**: each note
  spent takes one action, a bundle holds at most 16 actions (the
  `max_actions_per_bundle` parameter), and every transaction today carries one
  bundle (the protocol allows two a message). A payment from more than 16
  notes takes more than one transaction.
- **Transparent ERTH** is ordinary ERTH in an ordinary account, visible like on
  any other chain. It exists for the public parts of the economy: validators'
  self-bond, IBC transfers, exchanges, contracts, and payouts to the addresses
  of fund options.

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

## Liquidity is private too

LP shares are notes. Depositing into a pool from your shielded balance mints
your shares as a private note, and a withdrawal pays both sides back to you as
notes when it matures. The pool, its reserves and the amounts going in and out
are public, because a pool is shared money. Who deposited is not. LP share
notes can be sent only inside the pool: they cannot be unshielded.

## Registration has no wallet

Registering proves you hold a genuine passport. The proof is not tied to a
wallet address. It is tied to a **secret commitment**, a value derived from a
secret on your phone that the chain can check later proofs against without ever
seeing the secret.

The chain records each registration under its passport nullifier, the one-way
tag that stops one passport registering twice. That record holds the
registration time, the passport's country and its Document Signer certificate.
The registration also shows the referrer handle it named, if any. It holds no
address. Your registration reward and your first ANML are paid into notes that
only your phone can find.

### The passport nullifier can be recomputed

The passport nullifier is a fixed hash of data printed in the passport: the
issuing state, the document number, its check digit and the date of birth
(plus the optional-data field, where a long document number continues). It
has to be fixed. The same passport must give the same nullifier every time,
or one passport could register as many people. A value drawn from a secret on
your phone would change with the phone.

The cost is that anyone who holds that data can compute the nullifier and look
it up. The issuing state can. So can anyone with a copy or scan of the photo
page, such as a hotel, a border agency or a leaked database. They learn that
the passport registered, when, under which Document Signer certificate, and
which referrer handle it named.

It can also be guessed without the passport. Where a country's document
numbers are numeric or near-sequential, someone who knows your country and
date of birth can try candidate numbers until one matches a published
nullifier. The certificate on the registration narrows when the passport was
issued, and so the range to try. A correct guess reveals the same things.

They learn nothing past the registration. Nothing on chain links the
nullifier to a wallet, a balance, a claim, a Caretaker split, an assembly vote,
a handle you hold or anything you stake. Those use tags made from your secret
(below), and the passport data does not reach them. Timing can still hint at a
link: see [What can still leak](#what-can-still-leak).

From then on, everything you do as a person is a fresh proof: "I am one of the
registered humans, and here is a tag that stops me doing this twice". The proof
does not say which human.

### Switching phones or secrets

Registering the same passport again from **another wallet** while your
registration is live is a **switch**. (From the same wallet it is refused: see
[Renewing](./registering.md#renewing).) The old entry is retired, a new one
takes its place, and nothing is paid twice. Before you switch, the app can move your handle and your Caretaker
vote to the new identity, so neither has to wait. Without a move, the new
identity waits until anything the old one held has lapsed, because the chain
cannot tell the two apart from anyone else.

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

Some actions open only after a wait, so that one passport cannot act twice
under two secrets:

| Action | How often | Opens |
| --- | --- | --- |
| Claim 1 ANML | once a UTC day | the day after tomorrow (registering pays your first ANML at once) |
| Vote in the assembly | once a ballot, replaceable | at once, on every open ballot; after a switch or re-entry, only on ballots that open more than a day after it |
| Caretaker split | lasts a year, refresh it to keep it | at once for a new registration; after a switch, once the old identity's split could have lapsed, unless you moved it |
| Claim a handle | lasts a year, renew it to keep it | as for the Caretaker split |

A Caretaker split lapses unless refreshed, because the chain cannot find a
lapsed person's split to remove it. Nothing renews or refreshes on its own: the
app reminds you, and you confirm.

## Handles

A **handle** is a short name, like `@alice`, that a registered human claims for
their shielded address, so people can pay them without copying a long address.
One handle per person. A handle lasts a year. After that it is reserved to its
owner for a renewal window, and then it is free for anyone.

The handle directory is public: each handle and the address it names. Paying a
handle is private all the same. To pay `@alice`, the app downloads the **whole**
directory and looks her up on your phone, then checks the entry against the
chain's own copy before any money moves. It never asks a server for one handle,
so nobody learns which handle you looked up or paid.

## Referrals

A referral is named by handle. Open someone's link,
`https://erth.network/ref/<handle>`, or install the app from it, and the handle
is filled in for you at registration. You can remove or replace it.

When the registration lands, the chain splits the reward: your half to your own
private note, the referrer's half as a note to their handle's address. The
handle and the referral amount are public on the registration. Where the
referrer's note goes next is not.

## Private staking

Staking is private too. The private staking module, `x/shieldedstaking`, is
the only delegator to validators, apart from each validator's own self-bond.
You hold your share of its delegation as **stake notes**. They live in the
module's own note tree, beside the shielded pool's, not in the pool: you
delegate from the pool and are paid back into it.

- **One note per validator.** Delegating to a validator gives you **derth**,
  that validator's delegation token, as a private note. Staking more with the
  same validator merges into the note you already have, in the same
  transaction. A note's amount is never public.
- **Stake notes are owner-locked.** A stake proof can only create stake notes
  for the key that spent them, so derth cannot be sent or given to anyone
  else, and it is never a coin that could be traded. To hand stake to someone,
  unstake it and send the ERTH.
- **Exchange rate.** Each validator's derth has an exchange rate into ERTH.
  Rewards are restaked for everyone at once, so the rate rises. You never claim
  rewards: your derth becomes worth more ERTH.
- **Daily epochs.** Delegations and undelegations reach the validators together
  once a day, at the end of an epoch. That batches everyone's changes.
- **Unbonding** takes 21 days. When it ends the chain pays the ERTH into a
  private note for you by itself: there is nothing to claim. If the validator
  is slashed during unbonding, the payout is smaller, exactly as an ordinary
  unbonding would be.
- **Redelegation** moves stake to another validator at once, with no unbonding
  gap. The moved derth is **labelled** until the window in which the old
  validator can still be punished has closed, about the unbonding time. If the
  old validator is slashed in that window for something it did before you
  moved, the moved stake pays its share, not the new validator's other
  stakers. Until the label clears, the moved part cannot be unstaked or moved
  again. The rest of your stake moves freely.
- **Slashing** lowers the validator's exchange rate, so every holder of its
  derth shares the loss in proportion.

What is visible: the amounts entering or leaving each validator, and each
validator's rate. Not who holds the derth or how much.

Validators bond their own stake publicly from their operator address. Running a
validator is a public role. Ordinary delegation from a transparent account is
not possible on Earth.

### Voting with stake

To vote on a proposal with your stake, the app proves that your derth notes
existed, unspent, when the proposal opened. Nothing is spent, so the same notes
can vote on every open proposal. Each vote publishes a tag for that proposal,
so a note cannot vote twice on it, and a stake vote cannot be changed.

You cast **one vote per validator** you stake with. It carries one weight, the
sum of your notes there rounded down to three significant figures, so your
exact holdings do not show. The weight and the validator are public. The voter
is not. A validator's vote also covers any of its stake that did not vote, as
on other Cosmos chains.

### Groundworks positions

Voting in the Groundworks fund is a public act weighted by stake, so it uses a
**position**: derth locked into a public record with a split. The position
shows its size, its validator and its split. Its owner is proven by a tag that
only your phone can produce, and nothing links it to you. Locked derth keeps
earning. Unlock it and it merges back into your note at that validator.

## Fees: paid in ERTH, half burned

Every transaction pays a fee in ERTH. Every private transaction pays at least
the chain's minimum fee, which is part of consensus. A signed transaction pays
whatever minimum gas price the validators set on their nodes.

Private transactions are not signed by any account. They pay their fee from
your shielded ERTH, in the same transaction, with a proof that the fee came
from notes you own. Selling ANML therefore needs a little shielded ERTH for the
fee; your registration reward covers that.

Half of every fee is burned and half goes to validators.

Your first fee is covered by Earth's backend. Before you register, the app sends
the backend the registration it is about to broadcast. The backend checks it
with the chain's own rules and, if it is valid, shields a little ERTH into a new
note for you. It learns that a passport is about to register, which the chain
shows publicly anyway, and nothing about where the note goes next. After that,
your registration reward pays your fees. A switch is funded the same way,
because the new wallet starts empty. Either kind is granted once per passport
in any 30 days.

## Nothing happens without you

The app sends a transaction only when you confirm it. It never claims, refreshes,
renews or votes in the background, and never spends a fee you did not approve.
The daily ANML claim, the Caretaker refresh and the handle renewal are
reminders. The one thing that happens on its own is the chain paying out a
finished unbonding, and that costs you nothing.

## What can still leak

Privacy here is strong, not perfect. Know the limits.

- **A small crowd hides you less.** Your actions hide among everyone else's.
  At launch there are few registered humans and few notes, so a claim or a
  vote is one of only a handful. The privacy grows with the network. Staking
  changes are batched at epoch ends to help.
- **Timing.** If you shield 100 ERTH at noon and someone unshields 100 ERTH at
  12:01, an observer can guess. Wait between moving in and out, and avoid
  round, distinctive amounts.
- **Timing around your handle.** Registrations are public, and so is every
  handle being claimed or moved. If a handle is claimed shortly after a
  registration lands, or moved shortly before a switch lands, an observer can
  guess that the two belong together, and so link that passport's
  registration (its country, its signing certificate, and anyone who can
  recompute its nullifier, below) to the handle and the address it names. The
  fewer registrations there are, the better the guess. The app adds no delay
  of its own: it sends each step only when you tap, so the gap is yours to
  choose. Leave hours or days, not seconds, between registering and claiming
  a handle, and between moving a handle and switching.
- **Reusing a transparent address.** Everything a transparent ERTH address does
  is public and linked together. If you shield from and unshield to the same
  address, or post it with your name, it is not private. Use a fresh address
  for each public purpose.
- **Amounts at the edges.** Shielding, unshielding, delegating, undelegating,
  redelegating, swapping, providing liquidity and locking a position each
  reveal their amount and, where it applies, the pool or validator. Only the
  owner is hidden. An undelegation's payout, 21 days later, is linked to it.
- **Your handle.** A handle names your shielded address publicly. Payments to it
  stay private, but anyone who knows your handle knows that address is yours.
- **What the registration shows.** A registration shows the passport's country
  and its Document Signer certificate, which narrow down roughly which office
  issued it and when. With few registrants from one country, that is a small
  group.
- **Whoever holds your passport data.** Your issuing state, or anyone with a
  scan of your passport, can recompute your passport nullifier and see that
  you registered, when, under which certificate and with which referrer
  handle. Not your wallet or anything you do with it. See
  [above](#the-passport-nullifier-can-be-recomputed).
- **Your network connection.** Whoever relays your transaction or answers
  your app's queries sees your IP address. For Earth's own servers that
  includes your passport nullifier and your wallet address: see
  [What Earth's servers see](#what-earths-servers-see).

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

## What Earth's servers see

Your phone talks to three kinds of server Earth runs or uses: the backend (gas
for a first registration, the indexer, the handle directory, circuit
downloads), Earth's own chain node, and Cloudflare in front of both.

- **Gas.** The backend sees the registration or switch you are about to send,
  which the chain is about to publish anyway, including its passport
  nullifier and referrer handle, and pays a note it cannot follow. For each
  grant it stores the passport nullifier, the date, and whether it was a
  switch, and nothing that names the note or your wallet. It refuses a second
  grant to the same passport within any 30 days (a sliding window, not a
  calendar month), and deletes each record after 31 days, once it can no
  longer decide anything.
- **Indexer and handle directory.** The backend sees which ranges your phone
  downloads, like any website sees page requests. Every phone downloads the
  same ranges, and the whole directory.
- **Earth's node (lcd.erth.network, rpc.erth.network).** Unless you point the
  app at your own node, it sends every transaction you make through Earth's
  node and reads chain data from it. To show your transparent balances,
  liquidity withdrawals and transaction history, the app asks the node about
  your wallet's account address on each refresh. So the node receives your
  IP address, each transaction you broadcast, the address you ask about, and
  when.
- **Cloudflare.** The node and the backend are both reached through
  Cloudflare, which terminates the encrypted connection. Cloudflare receives
  everything the two servers receive: your IP address, the gas request with
  its passport nullifier, your transactions and your address queries.

Put together, this traffic could link your passport to your wallet: your IP
address arrives next to your passport nullifier when you ask for gas, and next
to your wallet address when the app refreshes or broadcasts. And anyone with
your passport data can recompute the nullifier (see
[above](#the-passport-nullifier-can-be-recomputed)).

**Earth's no-logs policy.** Earth's node and backend do not store client IP
addresses, request paths or gas-request identifiers. Rate limits use
short-lived counters held in memory only. The one record kept is the gas
grant's passport nullifier and date, above, for the 30-day limit. Cloudflare
is a separate company; what it records is governed by its own policy, not
ours.

**That is a promise, not a proof.** The app cannot show you that a server
keeps no logs. Whoever runs Earth's node and backend, and Cloudflare, could
see this traffic if they chose to, and you are trusting them not to. To trust
no one with it:

- **Run your own node** and point Earth Wallet at it (Settings → Network). Your
  transactions and address queries then go only to a machine you control. See
  [Use your own node](./run-a-node/wallet-node.md). The backend-only features
  still use Earth's backend: the handle directory, the private note streams,
  the gas grant and circuit downloads. All but the gas grant serve everyone
  the same data.
- **Use a VPN or Tor**, for registration especially (the gas request is the
  one that carries your passport nullifier) as well as for everyday use.

None of these servers ever receives passport data. The app contains no ads and
no advertising or tracking SDKs.
