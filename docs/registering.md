---
sidebar_position: 2
---

# Registering

You register one time, from your phone, with your passport.

1. The app reads the chip in the passport.
2. The phone builds a zero-knowledge proof. The proof shows that the passport is
   genuine and that a government the chain trusts signed it.
3. The app sends only the proof.

Your name, date of birth, passport number, and photo stay on the device. This is
a result of where the proof is generated. It is not a policy about the handling
of received data. The network does not receive the data.

## The nullifier

The chain records a **nullifier**. This is a one-way identifier that is derived
from the passport.

You cannot reverse a nullifier into a passport. The nullifier prevents a second
registration with the same passport. A renewed passport gives the same
nullifier, so a renewal does not create a second person.

This is the complete anti-Sybil mechanism. A second wallet does not create a
second account.

## Results of registration

- **1 ANML per day**, claimable in the app.
- **A vote in the Caretaker fund.** Each human has one vote. Holdings do not
  change it.
- **A share of the registration reward pool**, paid in ERTH at registration.

## Renewal

Registration is valid for one year. To renew, register again with the same
passport.

## Supported passports

The chain trusts the Country Signing Certificate Authorities that ICAO
publishes. It also trusts a small number of countries whose certificates ICAO
does not distribute. To add a country, use a governance proposal.

A passport without a readable chip does not work. This is a limit of the
document, not of the network.
