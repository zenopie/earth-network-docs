---
sidebar_position: 2
---

# Registering

You register once a year, from your phone, with your passport.

1. The app reads the chip in your passport over NFC.
2. The phone builds a zero-knowledge proof. It shows that the passport is
   genuine, that a government the chain trusts signed it, that it has not
   expired, and that you know the secret of the identity being registered,
   which only your phone holds.
3. The app asks Earth's backend for the fee. The backend checks the
   registration with the chain's own rules and pays a little shielded ERTH
   into a new private note for you.
4. The app sends the proof, paying the fee from that note. No wallet signs it.

Your name, date of birth, passport number and photo stay on the phone. That is
not a promise about how we handle your data: the data is never sent, so there is
nothing to handle. [Privacy](./privacy.md) lists exactly what the chain does
record.

## The nullifier

The chain records a **passport nullifier**, a one-way hash of the issuing
state, the document number, its check digit and your date of birth (plus the
passport's optional-data field, where a long document number continues). It
cannot be turned back into any of them. It is deterministic on purpose: the
same passport always gives the same nullifier, so a second registration with it
is recognised, and the chain refuses to count it as a new person.

Deterministic also means recomputable. Anyone who holds those fields can hash
them and look the nullifier up: the state that issued your passport, or anyone
with a scan of its photo page. They can see that the passport registered, when,
under which Document Signer certificate, and which referrer handle it named.
They cannot see your wallet, your balances, your claims or your votes: none of
those is tied to the passport nullifier. See
[Privacy](./privacy.md#the-passport-nullifier-can-be-recomputed).

It can also be guessed. Many countries issue document numbers that are
numeric or close to sequential, and the registration shows the country and
the Document Signer certificate, which narrows the time the passport was
issued and so the range its number falls in. Someone who knows your country
and date of birth can try candidate numbers, cheaply, until one matches a
published nullifier. A guess reveals the same things as a recomputation: that
the passport registered, when, its certificate and its referrer handle. Not
your wallet.

The proof is also bound to your **identity commitment**, a value derived from
your identity's secret (below), to the notes your rewards are paid into, to
your referrer's handle and to this network. Someone who copies your proof out
of a block cannot redirect it to themselves or replay it elsewhere.

The registration is tied to **no wallet address**. The chain records the
nullifier, the registration time, the passport's country and its Document
Signer certificate, and nothing that points at an account. Everything you do
as a person afterwards is a fresh proof that you are one of the registered
humans, without saying which. See [Privacy](./privacy.md#registration-has-no-wallet).

## Your identities

Each registration is to an **identity**: a secret the app derives from your
recovery phrase, and a public commitment to it that becomes your entry in the
chain's identity tree. Your ANML claims, votes, Caretaker split and handle are
all proven with that secret. Every registration proves that whoever sends it
knows the identity's secret, so nobody can register a passport to an identity
they did not create.

The chain accepts each identity **only once**. An identity that has
registered, whether it is still live, has lapsed or was switched away, is
refused if it is registered again, by any passport (error 1130). So one
recovery phrase holds a **series of identities**. The wallet's first
registration uses the first one, and every later registration from the same
wallet, a renewal or a fresh identity, uses its next one. Your notes, stake and
shielded address belong to the phrase, not to an identity, so they never move.
You never need a new wallet or a new recovery phrase to register again.

If the chain ever refuses a registration with 1130 (the app missed one of its
own registrations, for example after restoring from an older backup), the app
moves on to your next identity by itself. Start the registration again.

## What you get

- **1 ANML at once, then 1 ANML a day**, claimed privately when you choose to;
  the app reminds you. Days you do not claim do not carry over. Daily claims
  open the day after tomorrow.
- **A vote in the Caretaker fund**, one per human whatever you hold. It counts
  for a year at a time, and you refresh it when the app reminds you.
- **A handle**, a short name people can pay instead of your address, and your
  referral link.
- **A seat in the assembly**, the house that must approve every governance
  proposal. A first registration votes at once, on every open ballot. After a
  switch or a re-entry (below), you cannot vote on ballots that opened before
  it or within a day after it.
- **A share of the registration reward**, paid in shielded ERTH when you
  register. Each registration draws one ten-thousandth (1e-4) of the reward
  pool, half to you and half to your referrer, so each draw is a little
  smaller than the last and early registrants receive more. Caretaker votes
  pointed at the reward top the pool up. It pays your later fees.

The waits exist because the chain cannot follow you from one action to the
next. Without them, one passport could act twice, once under each of two
secrets.

## Referrals

A referrer is named by their **handle**. If you opened someone's link,
`https://erth.network/ref/<handle>`, or installed the app from it, the handle
is filled in for you; you can remove or replace it before you register. The app
checks that the handle is live before it is used.

The reward is then split: your half as a private note, the referrer's half as a
note the chain pays to their handle's address. The handle and the amount are
public on the registration. Without a referrer only your half is drawn (half
of 1e-4 of the pool), and the rest stays in the reward pool.

## Moving to a new phone

Restore your recovery phrase on the new phone. Your identities, your notes and
your registration come with it, because all of them derive from the phrase.

## Switching identity

Registering your passport again while its registration is live is a
**switch**: the old entry is retired, a new one takes its place, and nothing
is paid twice. The app's **Switch identity** screen offers two kinds of
target:

- **Another wallet** on the phone. The passport is registered to that wallet's
  next identity. Notes held by the old wallet stay with its phrase, so send
  them across first. Switching back to a wallet you used before works too: it
  registers its own next identity.
- **A fresh identity in this wallet**, from the same recovery phrase. Use it if
  you think this identity's secret alone was exposed. That is rare: the app
  derives the secret from your phrase and keeps it only in memory. It does not
  help if your **recovery phrase** may be exposed, because anyone with the
  phrase can derive every identity of the wallet. In that case create a new
  wallet, with a new phrase, and switch to it.

A passport can switch **once a day**. The switch must be proven on a later
date (UTC) than the registration it replaces, so a second switch the same day
is refused (error 1128); try again the next day. The app proves switches on
today's date.

If you lost the phrase, the old wallet's notes, handle and vote are lost with
it. Registering your passport from a new wallet is still a switch, but without
the old phrase nothing can be moved, so the new identity waits as below.

## Moving your handle and vote

Once a switch or a renewal has landed, the app can move your handle and your
Caretaker vote from the identity it replaced to the new one, so neither has to
wait. Open **Identity** to do it. A move is a private transaction proven with
both identities' secrets, which stay on the phone:

- From an earlier identity of the **same wallet** (a renewal, or a fresh
  identity), both secrets come from its one recovery phrase. The fee comes from
  the wallet's own private ERTH.
- From **another wallet**, that wallet must be on the phone too, with its
  phrase.

A move is **one step**: from an identity to the one that replaced it, and
only while the new one is your passport's live identity. After a second
switch, anything still held by the identity two steps back can never move;
it stays there until its lease ends. So move before you switch again. If the
identity you are leaving still holds something, the Switch identity screen
warns you, and switches only once you tick **Switch anyway and leave it where
it is**.

**When to move.** A move made right after a switch can be linked to it by its
timing (see [Privacy](./privacy.md#what-can-still-leak)), and a renewal is
just as public. So once the switch or renewal lands, the app picks a random
time between 6 hours and 3 days after it, and shows it as **Suggested: move
after** a date. Moving earlier is your choice (the app
asks you to confirm), and nothing hurries a move while the new identity stays
live. The app reminds you when the time comes. It never moves anything on its
own.

A handle in its renewal period cannot be moved, and once the switch has landed
the old identity can no longer renew it, so renew a handle that is close to
lapsing **before** you switch.

**Only to your own next identity.** The app proves, privately, that the old
identity and the new one follow each other under the same passport, without
saying which passport. And since every registration proves knowledge of its
identity's secret, every identity in your passport's chain is one you created
yourself. So a handle or a split cannot be sold to someone else's identity: a
buyer could at most be promised your secret, which you would still know. What
no passport-based system can stop is selling the passport itself. Handing
someone your passport's chip data, with the secret of your current identity,
lets them register your passport to their own identity and move your handle or
split there for one lease. That is selling your personhood, name and passport
number included.

Without a move, the old identity's handle keeps resolving and its split keeps
counting until their leases end, but neither can be renewed or changed, and
the new identity cannot claim a handle or cast a split until they could have
lapsed (up to a year).

## Renewing

A registration lasts one year. To renew it, open **Identity** once the year is
over and tap **Renew registration**. The app registers your passport to the
wallet's next identity, derived from the same recovery phrase: no new wallet
or phrase is needed. What happens depends on when, and to which identity:

| You register the same passport | Result |
| --- | --- |
| From the same wallet, after the registration has lapsed (**Renew registration**) | A **re-entry** with the wallet's next identity: a new registration, paid like the first (1 ANML and a share of the reward). |
| From another wallet, after the registration has lapsed | A re-entry with that wallet's next identity, paid the same way. |
| To a fresh identity in the same wallet, while the registration is live | A **switch** (above): the old entry is retired and nothing is paid. |
| From another wallet, while the registration is live | A switch to that wallet's next identity. |
| To an identity that has registered before | Refused: error 1123 while that registration is live, 1130 after. The app never sends one. |

A re-entered identity is bound by its predecessor, as a switched one is,
because the chain cannot tell whether the lapsed identity and the new one are
the same person. It cannot vote on assembly ballots that opened before the
re-entry or within a day after it. It cannot claim a new handle or cast a new
Caretaker split until anything the old identity could have held has lapsed.
A handle or split your previous identity still holds live does not need that
wait: bring it over from Identity (above). The previous identity can no longer
renew it, so move it before its lease ends. Daily ANML claims open the day
after tomorrow, as for a first registration.

### A new passport

A renewed passport has a new passport number, so it produces a **new**
nullifier. This is deliberate: a nullifier built from something that never
changes, like your name, could be computed by anyone who knows your name and
birthday, for the rest of your life. The document number at least limits that
to holders of this passport's data, or to someone able to guess its number
(above), and only until the passport is replaced. The cost is that for a short time one person can hold a registration from
the old passport and one from the new. The old one lapses at the end of its
year.

The same applies to anyone who holds two valid passports at once, such as dual
nationals. Earth treats each passport as one registration.

## Who can register

You need a passport with a readable chip, issued by a country whose signing
certificates the chain trusts. At launch the chain trusts the certificates in
ICAO's master list and nothing else. Adding a country, including one ICAO does
not distribute, takes a governance proposal.

Many people in the world do not hold a chip passport. They cannot register yet.
That is a real limit of building on passports, and it is the price of not
needing a company, a biometric scanner or a database to decide who is a person.

## Before you register

- Use the latest app. After an upgrade that changes the proof, only proofs from
  the current app verify.
- Write down your recovery phrase. It is the only backup of your registration
  secret and your private balances.
