---
sidebar_position: 2
---

# Registering

You register once a year, from your phone, with your passport.

1. The app asks Earth's backend for the fee. The backend checks the
   registration is valid and pays a little shielded ERTH into a new private
   note for you.
2. The app reads the chip in your passport over NFC.
3. The phone builds a zero-knowledge proof. It shows that the passport is
   genuine, that a government the chain trusts signed it, that it has not
   expired, and that the registration is bound to a secret only your phone
   holds.
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
secret on your phone, and to the notes your rewards are paid into. Someone who
copies your proof out of a block cannot redirect it to themselves.

The registration is tied to **no wallet address**. The chain records the
nullifier, the registration time, the passport's country and its Document
Signer certificate, and nothing that points at an account. Everything you do
as a person afterwards is a fresh proof that you are one of the registered
humans, without saying which. See [Privacy](./privacy.md#registration-has-no-wallet).

## What you get

- **1 ANML at once, then 1 ANML a day**, claimed privately by the app. Days you
  do not claim do not carry over. Daily claims open the day after tomorrow.
- **A vote in the Caretaker fund**, one per human whatever you hold. It opens
  30 days after you register.
- **A seat in the assembly**, the house that must approve every governance
  proposal. You can vote on every ballot that opens after you register.
- **A share of the registration reward**, paid in shielded ERTH when you
  register. It halves as more people register, so early registrants receive
  more. It pays your later fees.

The waits exist because the chain cannot follow you from one action to the
next. Without them, one passport could act twice, once under each of two
secrets.

## Referrals

If someone referred you, the app names their **referral address**, and the
reward is split: your half as a private note, theirs in public ERTH to that
address. A referral address has to be bound by a registered human first, with
a proof that they are one, so the chain knows a real person vouches for it, not
who. A registered human can bind one address, 30 days after registering, and
the app keeps the binding fresh. Without a referrer you receive your half and
the rest stays in the reward pool.

## Moving to a new phone

Restore your recovery phrase on the new phone. Your secret, your notes and your
registration come with it, because all of them derive from the phrase.

If you lost the phrase, or think your secret is exposed, scan the same passport
again from a new wallet. While your registration is live this is a **switch**:
the old entry is retired, a new one takes its place, and nothing is paid twice.
A switch restarts the waits above, and notes held by the old wallet stay with
the old phrase.

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
