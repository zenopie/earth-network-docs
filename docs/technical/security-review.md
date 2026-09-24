---
sidebar_position: 1
title: Security review
---

# Security review, September 2026

A full review of every Earth Network repository on 2026-09-23 found two critical
problems. Both are fixed and ship in chain upgrade `v0.9.1` and the current apps.
Every high-severity problem in the chain and the apps is fixed too. This page
lists what shipped and everything still open, so anyone can check our progress.

The review covered the chain, the circuits, both mobile apps, the web app, the
backend, the deployment and these docs. Every critical and high finding was
checked against the code before it was fixed. Each fix has a test, and each
critical was reproduced before it was fixed.

## Fixed

`v0.9.1` is governance proposal 6. Both houses approved it, and it applies at
height **467,500**. The circuit fix needs the matching app: from that height,
only proofs from the current app verify.

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

## Open

Nothing open in the chain, the apps or the backend is critical or high. The
three high items left are in the deployment and are being handled separately.

### Chain

| Severity | Problem | Ships via |
| --- | --- | --- |
| Medium | The assembly refunds a failed proposal's deposit even when stake vetoed it or it was spam, which removes the deterrent x/gov's burn provides | Next upgrade |
| Medium | Striking a Groundworks option runs in EndBlock without isolation, so an error there would halt the chain | Next upgrade |
| Medium | Certificate verification accepts a CA certificate, or a self-signed one, in place of a Document Signer | Next upgrade |
| Medium | The per-country registration cap reads the country from the signer's own certificate and skips an empty one; the network-wide cap is counted but never enforced | Next upgrade |
| Medium | The gas charged for verifying a proof is probably below its real CPU cost, so blocks of invalid proofs are cheap | Governance parameter |
| Medium | The proof verifier accepts field elements of p or more and does not pin the number of public inputs (registration already guards against both) | Node software |
| Medium | The verifier library's headers are fetched by a movable git tag and the built library has no pinned hash | Build change |
| Medium | Validators download upgrade binaries automatically and the node container runs as root | Image change |
| Low | The nullifier does not include the issuing country, and a document number longer than 9 characters is truncated | Circuit change |
| Low | Interchain accounts may execute any message type | Governance parameter |
| Low | Moving a registration to a new wallet uses up the signer's and country's daily allowance | Next upgrade |
| Low | Parameter validation allows an unbounded registration validity and non-distinct public-input indexes | Next upgrade |
| Low | Staking emission catch-up after a genesis import is uncapped | Next upgrade |
| Low | A proposal cancelled while in voting leaves its assembly ballot open | Next upgrade |
| Low | Exchange genesis validation misses duplicate token denoms and bids that do not add up | Next upgrade |
| Low | Re-seeding a pool with no shares hands the depositor any reserve left in it | Next upgrade |
| Low | After a governance reset of the Groundworks slate, a staking change revives a voter's old split | Next upgrade |
| Low | The LP unbondings query walks every unbonding on the chain | Node software |

### Mobile apps

| Severity | App | Problem |
| --- | --- | --- |
| Medium | Both | The transaction confirmation does not show the recipient or the minimum you will receive |
| Medium | Both | The unlock lockout can be shortened by moving the device clock |
| Medium | Android | An imported recovery phrase is not checksum-validated, so a typo opens an empty wallet |
| Medium | Android | The recovery phrase field lets the keyboard learn and autocorrect the words |
| Medium | Android | Any Keystore error deletes the biometric unlock, not only an enrolment change |
| Medium | Android | Phone-to-phone transfer is not excluded |
| Medium | iOS | The app-switcher privacy cover does not reach sheets, so a revealed phrase can appear in the snapshot |
| Medium | iOS | Saving the vault deletes the old copy before writing the new one |
| Low | Both | A transaction that times out waiting for confirmation is shown as a success |
| Low | Both | The network's minimum gas price is accepted without a ceiling |
| Low | Android | The passport screens are not protected from screenshots |
| Low | Android | BouncyCastle 1.70 has known denial-of-service CVEs |

### Backend (rewarded-ad payouts)

| Severity | Problem |
| --- | --- |
| Medium | The callback signature is checked against the URL-decoded query, but parameters are parsed from the raw one. A percent-encoded `&` in a captured callback yields a new transaction id with a valid signature, so one callback can be paid more than once |
| Medium | There is no per-address or daily payout limit, so the payout wallet can be drained by farming ad views |
| Medium | The chain client has no HTTP timeout, so one hung request stops all payouts |
| Low | The callback timestamp is not checked; the tunnel image is unpinned; the container runs as root |

### Web app

| Severity | Problem |
| --- | --- |
| Medium | A slow quote can overwrite a newer one, weakening a swap's minimum output |
| Medium | The liquidity deposit floor becomes zero when the pool's share supply fails to load |
| Medium | Transaction errors are never shown to the user |
| Medium | No HSTS header |
| Low | Amounts are parsed as floating point; slippage is not clamped; `..` in an address path is not rejected |

### Deployment

| Severity | Problem |
| --- | --- |
| High | A genesis-reset switch is still set on the validator: an image with a different genesis file would wipe its history |
| High | The validator operator account's mnemonic was passed to earlier hosting providers and should be treated as exposed |
| High | The lease-log script sends a shell-capable token without verifying TLS |
| Medium | The tunnel token needs rotating |
| Medium | The validator's P2P address is not published, so new nodes cannot peer |

### Keys and docs

| Severity | Problem |
| --- | --- |
| Medium | Old API keys appear in the public git history of the backend and web app repositories and must be revoked |
| Low | The docs site's build dependencies have five high `npm audit` findings |
