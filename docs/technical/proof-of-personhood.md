---
sidebar_position: 2
title: Proof of personhood
---

# Proof of personhood: how it works now

Updated 2026-09-24, after the v0.9.1 circuit fix. Registration is live on
earth-1: real passports have been scanned on real phones and registered, so
on-device proving is proven, not pending.

## The stack

- **Circuits:** ours, in `earth-network-mobile/circuits/`. `poa_core` holds the
  shared logic (hash binding, expiry, address binding, nullifier, DSC
  commitment). Seven `lean_poa_*` binaries differ only in how they verify the
  SOD signature: P-256, P-384, brainpool 256/384/512, RSA-2048, RSA-4096. The
  approach and the proving library come from zkPassport; the circuits do not.
- **Proof system:** Noir with Barretenberg UltraHonk, poseidon2 flavour, no
  trusted setup. Everything is pinned to **bb v5.0.0 final**. Android proves
  with `noir_android 1.0.0-beta.22-2`, iOS with Swoirenberg, and the chain
  verifies with `zk/ultrahonk`, CGo bindings over the v5.0.0 verifier
  (`third_party/barretenberg-go`). A nightly bb changes the transcript and the
  chain rejects its proofs, so never float these pins.
- **Compile:** `nargo 1.0.0-beta.22`. The compiled circuits are checked in at
  `android/app/src/main/assets/circuits/`, and iOS bundles the same files at
  build time. An old app build proves against old keys and fails after a key
  rotation.

## What the circuit proves

1. `dg1_len == 93` and DG1 starts `61 5B 5F 1F 58`: a whole TD3 data page.
2. `sha256(DG1)` sits inside the hashed eContent, right after DG1's
   DataGroupHash DER (`30 25 02 01 01 04 20`).
3. `sha256(eContent)` sits inside the hashed signed attributes, right after the
   messageDigest attribute DER (`06 09 2A864886F70D010904 31 22 04 20`).
4. The DSC signature over `sha256(signedAttrs)` verifies under the DSC key.
5. The passport has not expired as of `current_date`.
6. `address != 0`. The address is a public input, so the proof verifies only for
   the wallet it was made for.

Points 1 to 3 are the v0.9.1 fix. Before it, an embedded hash was bounded only by
its buffer, and `sha256_var` ignores bytes past its length, so a genuine SOD could
carry an invented DG1's hash in its unhashed tail. See the [security review](./security-review.md).

## Public inputs

Four field elements, in this order, and the chain's params agree:

| Index | Value | Param |
| --- | --- | --- |
| 0 | `current_date` (YYMMDD) | `current_date_index` |
| 1 | registrant address | `address_index` |
| 2 | nullifier | `nullifier_index` |
| 3 | DSC commitment | `dsc_key_index` |

The chain rejects any input of p or more before verifying, compares the address
to the signer as an integer, and pins `current_date` to block time within the
skew parameter.

## Nullifier

`Poseidon2(document number ‖ DOB)`, 15 field elements. It was name ‖ DOB until
the address-binding change. That was renewal-stable, but anyone with a name and
birthday could confirm which wallet was that person's. The cost now is that a
renewed passport, or a second passport, gives a second registration until the
first lapses.

Still open, and batched for the next circuit change:

- The issuing state is not in the preimage, so two countries' identical
  9-character numbers with the same DOB collide, and the second registrant
  displaces the first.
- A document number over 9 characters overflows into the optional-data field and
  is truncated.

## DSC commitment

`Poseidon2(curve tag ‖ canonical key bytes)`. The tag separates same-width curves
(P-256 from brainpool256, P-384 from brainpool384). `CURVE_TAG_*` in `poa_core`
must equal `certs.Tag*` in the chain for ever: append only, never renumber. The
chain recomputes the commitment from the DSC certificate in the transaction and
requires it to equal public input 3. So the circuit cannot sign with one key and
name another.

## Trust store

`x/pki` holds the CSCA certificates, one record per certificate (not per key),
seeded in genesis from `csca/`. A registration carries its DSC certificate, which
must chain to a trusted, unrevoked CSCA and must not itself be revoked. The old
`dsc_root` Merkle registry is gone; its param field is `reserved`.

Known gap: `VerifyDsc` does not reject a CA or self-signed certificate
presented as the DSC. See the [security review](./security-review.md).

## Verifying keys, by era

| From | Keys | Where |
| --- | --- | --- |
| Genesis | Original seven | `networks/genesis/verifying-keys/`, baked into genesis.json. Never regenerate: replay needs them |
| v0.7.0 (height 50,100) | Curve tag added to the commitment | `app/upgrades/v070/verifying-keys/` |
| v0.9.1 (height 467,500) | Bounded hash binding | `app/upgrades/v091/verifying-keys/` |

Each rotation is done in the upgrade handler, not by a params proposal, so the
keys change at exactly the height the binary changes. `swapVerifyingKeys` refuses
unless the embedded set and the params name the same seven circuits.

## Regenerating after a circuit change

1. `nargo compile --workspace` in `circuits/`, then copy `target/*.json` into
   the Android assets.
2. In the chain repo, update `tools/poafixtures` if the witness shape changed,
   then run `scripts/regen-poa-fixtures.sh ../earth-network-mobile`. It proves
   every variant and refreshes the fixtures in `zk/ultrahonk/testdata` and
   `x/personhood/keeper/testdata`.
3. Base64 each `zk/ultrahonk/testdata/<variant>/vk` into
   `app/upgrades/<version>/verifying-keys/<variant>.vk.b64`, with no trailing
   newline.
4. `TestV091EmbedsTheRecompiledVerifyingKeys` is the pattern for the check that
   embedded keys equal the fixtures. Copy it for the next version.
5. Ship the app build before the upgrade height.
