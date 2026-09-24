---
sidebar_position: 2
---

# Registering

You register once a year, from your phone, with your passport.

1. The app reads the chip in your passport over NFC.
2. The phone builds a zero-knowledge proof. It shows that the passport is
   genuine, that a government the chain trusts signed it, that it has not
   expired, and that the registration is for your wallet address.
3. The app sends only the proof.

Your name, date of birth, passport number and photo stay on the phone. That is
not a promise about how we handle your data: the data is never sent, so there is
nothing to handle. [Privacy](./privacy.md) lists exactly what the chain does
record.

## The nullifier

The chain records a **nullifier**, a one-way tag computed from your passport
number and date of birth. It cannot be turned back into either. A second
registration with the same passport produces the same nullifier, and the chain
refuses it as a new person.

The proof is also bound to the wallet address you register from. Someone who
copies your proof out of a block cannot use it for their own wallet.

## What you get

- **1 ANML a day**, claimed in the app. Days you do not claim do not carry over.
- **A vote in the Caretaker fund**, one per human whatever you hold.
- **A seat in the assembly**, the house that must approve every governance
  proposal.
- **A share of the registration reward**, paid in ERTH when you register. It
  halves as more people register, so early registrants receive more. If someone
  referred you, you and they each receive a share.

## Moving to a new wallet

Register again from the new wallet with the same passport. The chain recognises
the nullifier and moves your registration, with its ANML clock, to the new
address. There is no second reward and no second vote.

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
- Register from the wallet you intend to keep. You can move later, but it takes
  another scan.
