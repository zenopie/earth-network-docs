---
sidebar_position: 2.5
---

# Privacy

Registering proves you are a person without telling the chain who you are. The
proof is built on your phone, and your passport data is never sent anywhere.
But a registration is a public record, and it does say some things about you.
This page lists both.

## What never leaves your phone

- Your name
- Your date of birth
- Your passport number
- Your photo, and everything else on the passport's data page

The app reads these from the chip to build the proof and then discards them. They
are not sent to the chain, to Earth's backend, or to anyone else.

## What the chain records, publicly

Everything on a blockchain can be read by anyone, for ever. A registration
records:

| Field | What it is | What it reveals |
| --- | --- | --- |
| Wallet address | The account you registered from | That this address belongs to a registered human |
| Nullifier | A one-way tag from your passport number and date of birth | Nothing on its own. It cannot be reversed, and it cannot be guessed without your passport number |
| Issuing country | The three-letter code of the country that issued your passport | Your nationality, or one of them |
| Document Signer | The certificate that signed your passport | Roughly which office and period issued it. Countries rotate these every few months |
| Registration time | When you registered | When you registered |
| Last ANML claim | The day you last claimed | How active you are |

The register transaction itself also contains the Document Signer certificate and
the proof. Neither contains anything beyond the fields above.

## What this means in practice

- Anyone can see that your wallet belongs to a registered human, and from which
  country. If you want your registration kept apart from your other activity,
  register from a fresh wallet and use it only for that.
- Nobody can go from your wallet to your name through the chain. Linking them
  would take information from outside it, such as you posting both.
- Because the nullifier uses your passport number, knowing your name and birthday
  is not enough to find your wallet. See [Registering](./registering.md#renewing)
  for what that choice costs.

## What Earth's backend sees

The app talks to a small backend only for rewarded ads, which pay for a new
wallet's first fees. That service sees the wallet address it pays, and the ad
network sees what any ad network sees. It never sees passport data.
