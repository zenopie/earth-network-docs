---
sidebar_position: 2
---

# Registering

You register once a year, from your phone, with your passport.

1. The app reads the chip in your passport over NFC.
2. The phone builds a zero-knowledge proof. It shows that the passport is
   genuine, that a government the chain trusts signed it, that it has not
   expired, and that the registration is bound to a secret only your phone
   holds.
3. The app asks Earth's backend for the fee. The backend checks the
   registration with the chain's own rules and pays a little shielded ERTH
   into a new private note for you.
4. The app sends the proof, paying the fee from that note. No wallet signs it.

Your name, date of birth, passport number and photo stay on the phone. That is
not a promise about how we handle your data: the data is never sent, so there is
nothing to handle. [Privacy](./privacy.md) lists exactly what the chain does
record.

## The nullifier

The chain records a **nullifier**, a one-way tag computed from your passport
number and date of birth. It cannot be turned back into either. A second
registration with the same passport produces the same nullifier, and the chain
refuses it as a new person.

The proof is also bound to your **secret commitment**, a value derived from a
secret on your phone, to the notes your rewards are paid into, to your
referrer's handle and to this network. Someone who copies your proof out of a
block cannot redirect it to themselves or replay it elsewhere.

The registration is tied to **no wallet address**. The chain records the
nullifier, the registration time, the passport's country and its Document
Signer certificate, and nothing that points at an account. Everything you do
as a person afterwards is a fresh proof that you are one of the registered
humans, without saying which. See [Privacy](./privacy.md#registration-has-no-wallet).

## What you get

- **1 ANML at once, then 1 ANML a day**, claimed privately when you choose to;
  the app reminds you. Days you do not claim do not carry over. Daily claims
  open the day after tomorrow.
- **A vote in the Caretaker fund**, one per human whatever you hold. It counts
  for a year at a time, and you refresh it when the app reminds you.
- **A handle**, a short name people can pay instead of your address, and your
  referral link.
- **A seat in the assembly**, the house that must approve every governance
  proposal. You can vote on every ballot that opens after you register.
- **A share of the registration reward**, paid in shielded ERTH when you
  register. It halves as more people register, so early registrants receive
  more. It pays your later fees.

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
public on the registration. Without a referrer you receive your half and the
rest stays in the reward pool.

## Moving to a new phone

Restore your recovery phrase on the new phone. Your secret, your notes and your
registration come with it, because all of them derive from the phrase.

If you think your secret is exposed, scan the same passport again from a new
wallet. While your registration is live this is a **switch**: the old entry is
retired, a new one takes its place, and nothing is paid twice. Before
switching, move your handle and your Caretaker vote to the new wallet from the
old one; otherwise the new identity waits until they could have lapsed. Notes
held by the old wallet stay with the old phrase, so send them across first.

If you lost the phrase, the old wallet's notes, handle and vote are lost with
it. Scanning your passport from a new wallet is still a switch, but nothing can
be moved, so the new identity waits as above.

## Renewing

A registration lasts one year. To renew, register again.

A renewed passport has a new passport number, so it produces a **new**
nullifier. This is deliberate: a nullifier built from something that never
changes, like your name, could be guessed by anyone who knows your name and
birthday, and they could then find your wallet. The cost is that for a short
time one person can hold a registration from the old passport and one from the
new. The old one lapses at the end of its year.

The same applies to anyone who holds two valid passports at once, such as dual
nationals. Earth treats each passport as one registration.

## Who can register

You need a passport with a readable chip, issued by a country whose signing
certificates the chain trusts. The chain trusts the certificates that ICAO
publishes, plus a few countries that ICAO does not distribute. Adding a country
takes a governance proposal.

Many people in the world do not hold a chip passport. They cannot register yet.
That is a real limit of building on passports, and it is the price of not
needing a company, a biometric scanner or a database to decide who is a person.

## Before you register

- Use the latest app. After an upgrade that changes the proof, only proofs from
  the current app verify.
- Write down your recovery phrase. It is the only backup of your registration
  secret and your private balances.
