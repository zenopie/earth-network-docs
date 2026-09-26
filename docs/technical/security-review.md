---
sidebar_position: 1
title: Security review
---

# Security review, September 2026

A full review of every Earth Network repository on 2026-09-23 found two critical
problems. Both are fixed and shipped in chain upgrade `v0.9.1`. Every other
finding in the chain, the circuits, the apps, the web app and the backend is now
fixed as well. What is left is in the deployment, and old API keys to revoke.
This page lists what was fixed and what is still open, so anyone can check.

The review covered the chain, the circuits, both mobile apps, the web app, the
backend, the deployment and these docs. Every critical and high finding was
checked against the code before it was fixed. Each fix has a test, and each
critical was reproduced before it was fixed.

## Releases

- **`v0.9.1`** — governance proposal 6. Both houses approved it, and it applied
  at height **467,500**.
- **`v0.9.2`** — governance proposal 7, in its voting period. If both houses
  approve it, it applies at height **505,000**. It changes the nullifier, so it
  needs the matching app (iOS build 5, Android 1.0.28), and it retires every
  existing registration at that height: each person registers again, once.
- **Web app and backend** fixes are merged and ship with their next deploy.

## Fixed in `v0.9.1` and the apps that shipped with it

| Severity | Problem | Fix | Ships in |
| --- | --- | --- | --- |
| Critical | The register circuit checked that each embedded hash fitted inside its fixed-size buffer, not inside the bytes that were actually hashed. One genuine passport could register unlimited fake people. | Each hash must sit inside the hashed bytes, directly after its expected ASN.1 header | Circuits and new verifying keys, `v0.9.1` |
| Critical | Allocation rounded its reserve down, while options that settle rarely collect the rounding. The first address option with votes could eventually halt the chain. | Round the reserve up | `v0.9.1` |
| High | Any short data group could stand in for the passport's main data page | The circuit requires a whole 93-byte TD3 data page with its header | Circuits |
| High | The exchange's volume index grew without limit: LP rewards would stall within a year, and an overflow would halt the chain in about 3.6 years | The index rebases itself about every six months | `v0.9.1` |
| High | Exporting and re-importing the exchange's state dropped pending LP rewards, so the first block after an import would halt | Genesis carries pending rewards, the volume index and the staleness queue | `v0.9.1` |
| High | Registrations under a revoked Document Signer kept voting until they were purged, including on their own revocation | They stop voting at once, cannot vote on their own revocation, and retired registrations' votes come off open ballots | `v0.9.1` |
| High | Closing an assembly ballot did work proportional to its turnout in a single block | Each round of voting is its own ballot; closing is constant-time and cleanup is capped per block | `v0.9.1` |
| High | Android stored an unsalted hash of the unlock PIN beside the wallet, and never relocked | The PIN is checked only by decrypting; the app locks after 60 seconds away and on screen-off | Android app |
| High | iOS showed the recovery phrase and changed the unlock method without asking again, never auto-locked, let the device passcode open the biometric slot, and derived a different wallet from a phrase typed in capitals | Re-authentication for both, 60-second auto-lock, biometrics bound to current enrolment, phrases normalised | iOS app, TestFlight build 4 |
| Medium | Exchange LP rewards could pay out slightly more than was paid in | Round the reserve up | `v0.9.1` |

## Fixed in `v0.9.2`

| Severity | Problem | Fix |
| --- | --- | --- |
| Medium | The assembly refunded a failed proposal's deposit even when stake vetoed it or it was spam | The deposit is burned whenever stake's own tally would burn it |
| Medium | Striking a Groundworks option ran in EndBlock without isolation, so an error there would halt the chain | The strike runs isolated; a failure is rolled back and reported |
| Medium | Certificate verification accepted a CA certificate, or a self-signed one, in place of a Document Signer | A certificate that is a CA, may sign certificates or is self-issued is refused |
| Medium | The per-country registration cap read the country from the signer's own certificate and skipped an empty one; the network-wide cap was counted but never enforced | The country comes from the trusted issuer, an empty one is capped too, and the network has a daily cap |
| Medium | The gas charged for verifying a proof was below its real CPU cost | Proof verification 3,000,000 gas, certificate verification 300,000 |
| Medium | The proof verifier accepted field elements of p or more and did not pin the number of public inputs | Both refused |
| Medium | The verifier library's headers were fetched by a movable git tag | Headers and their dependency are pinned by commit and checked (node image) |
| Medium | The node container ran as root | It runs as an unprivileged user (node image) |
| Low | The nullifier did not include the issuing country, and a document number longer than 9 characters was truncated | The nullifier covers the issuing state and the whole document number; new circuits and verifying keys |
| Low | Interchain accounts could execute any message type | An explicit list of allowed messages |
| Low | Moving a registration to a new wallet used up the signer's and country's daily allowance | A move is neither limited nor counted |
| Low | Parameter validation allowed an unbounded registration validity and non-distinct public-input indexes | Validity is bounded to three years; indexes must differ |
| Low | Staking emission catch-up after a genesis import was uncapped | Emission clocks resume at the genesis time |
| Low | A proposal cancelled while in voting left its assembly ballot open | Its ballot is closed |
| Low | Exchange genesis validation missed duplicate token denoms and bids that did not add up | Both refused |
| Low | Re-seeding a pool with no shares handed the depositor any reserve left in it | The leftover is burned first |
| Low | After a governance reset of the Groundworks slate, a staking change revived a voter's old split | A split from before the reset is dropped |
| Low | The LP unbondings query walked every unbonding on the chain | It reads an index by address |

Validators download upgrade binaries automatically when a governance upgrade
passes. That is deliberate and stays: the download must match the checksum in
the proposal, the proposal needs both stake and two thirds of human votes, and
an operator who does not want automatic upgrades can run without cosmovisor.

## Fixed in the apps (iOS build 5, Android 1.0.28)

| Severity | App | Problem | Fix |
| --- | --- | --- | --- |
| Medium | Both | The transaction confirmation did not show the recipient or the minimum you will receive | Both are shown, the address in full |
| Medium | Both | The unlock lockout could be shortened by moving the device clock | Timed on a clock that cannot be set, surviving reboots |
| Medium | Android | An imported recovery phrase was not checksum-validated | The word list and checksum are checked |
| Medium | Android | The recovery phrase field let the keyboard learn and autocorrect the words | No suggestions and no learning |
| Medium | Android | Any Keystore error deleted the biometric unlock | Only an enrolment change does |
| Medium | Android | Phone-to-phone transfer was not excluded | All app data is excluded from backup and transfer |
| Medium | iOS | The app-switcher privacy cover did not reach sheets | The cover is a window above everything |
| Medium | iOS | Saving the vault deleted the old copy before writing the new one | Replaced in one operation |
| Low | Both | A transaction that timed out waiting for confirmation was shown as a success | Shown as not confirmed yet, with its hash |
| Low | Both | The network's minimum gas price was accepted without a ceiling | Above 0.1uerth it is ignored |
| Low | Android | The passport screens were not protected from screenshots | Protected |
| Low | Android | BouncyCastle 1.70 has known denial-of-service CVEs | BouncyCastle 1.86 |

## Fixed in the backend and the web app

| Severity | Component | Problem | Fix |
| --- | --- | --- | --- |
| Medium | Backend | The callback's parameters were parsed from the raw query, not the signed one, so a re-encoded callback could be paid twice | Parameters are read from the signed text |
| Medium | Backend | No per-address or daily payout limit | 5 per address and 1,000 in total per 24 hours |
| Medium | Backend | The chain client had no HTTP timeout | 15 seconds, and a timeout after broadcast is never paid twice |
| Low | Backend | The callback timestamp was not checked; the tunnel image was unpinned; the container ran as root | Timestamp window, image pinned by digest, unprivileged user |
| Medium | Web app | A slow quote could overwrite a newer one | Only the latest quote is used |
| Medium | Web app | The liquidity deposit floor became zero when the pool's share supply failed to load | The deposit is refused instead |
| Medium | Web app | Transaction errors were never shown | The reason is shown |
| Medium | Web app | No HSTS header | Sent on every response |
| Low | Web app | Amounts parsed as floating point; slippage not clamped; `..` accepted in an address path | Exact decimal parsing, slippage clamped to 0.1–50%, `..` refused |
| Low | Docs | The docs site's build dependencies had five high `npm audit` findings | Cleared |

## Open

| Severity | Area | Problem |
| --- | --- | --- |
| High | Deployment | A genesis-reset switch is still set on the validator: an image with a different genesis file would wipe its history |
| High | Deployment | The validator operator account's mnemonic was passed to earlier hosting providers and should be treated as exposed |
| High | Deployment | The lease-log script sends a shell-capable token without verifying TLS |
| Medium | Deployment | The tunnel token needs rotating |
| Medium | Deployment | The validator's P2P address is not published, so new nodes cannot peer |
| Medium | Keys | Old API keys appear in the public git history of the backend and web app repositories and must be revoked |
