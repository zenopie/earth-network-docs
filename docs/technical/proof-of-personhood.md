---
sidebar_position: 2
title: Proof of personhood
---

# Proof of personhood: how it works

A registration is one zero-knowledge proof, made on the phone from the
passport's chip and checked by the chain. It proves the passport is genuine
and that the registrant knows the secret of the identity it registers. It
names no wallet.
Every later personal action (ANML claims, assembly votes, Caretaker splits,
handles) is a separate **membership** proof against the chain's identity
tree. See [Privacy](../privacy.md).

## The stack

- **Circuits:** ours, in `earth-network-mobile/circuits/`. `poa_core` holds the
  shared logic (hash binding, expiry, the binding input, nullifier, DSC
  commitment). 33 `lean_poa_*` register circuits differ only in the DSC key
  type, signature padding and hash profile: RSA-2048/3072/4096 with PKCS#1
  v1.5 or PSS and any exponent in [3, 2^17), and ECDSA on P-224, P-256, P-384,
  P-521 and brainpoolP224r1/256r1/384r1/512r1, each with the SHA-1 to SHA-512
  hash combinations real passports carry. The message's `signature_algorithm`
  names the variant, and genesis carries one verifying key per variant. The
  approach and the proving library come from zkPassport; the circuits do not.
- **Proof system:** Noir with Barretenberg UltraHonk, poseidon2 flavour, no
  trusted setup. Everything is pinned to **bb v5.0.0 final**. Android proves
  with `noir_android 1.0.0-beta.22-2`, iOS with Swoirenberg, and the chain
  verifies with `zk/ultrahonk`, CGo bindings over the v5.0.0 verifier. A
  nightly bb changes the transcript and the chain rejects its proofs, so
  never float these pins.
- **Compile:** `nargo 1.0.0-beta.22`. The compiled circuits are checked in at
  `android/app/src/main/assets/circuits/`, and iOS bundles the same files at
  build time. An app built before a key change proves against old keys and
  fails.

## What the circuit proves

1. `dg1_len == 93` and DG1 starts `61 5B 5F 1F 58`: a whole TD3 data page.
2. `sha256(DG1)` sits inside the hashed eContent, right after DG1's
   DataGroupHash DER (`30 25 02 01 01 04 20`).
3. `sha256(eContent)` sits inside the hashed signed attributes, right after
   the messageDigest attribute DER (`06 09 2A864886F70D010904 31 22 04 20`).
4. The DSC signature over `sha256(signedAttrs)` verifies under the DSC key.
5. The passport has not expired as of `current_date`.
6. The binding input is not zero.
7. `idc = H(TAG_ID, id_secret)`, output as public input 4, where `id_secret`
   is a private witness: the registrant knows the secret of the identity it
   registers.

Each embedded hash must sit inside the bytes that were actually hashed, not
merely inside its buffer: `sha256_var` ignores bytes past its length, so a
looser check would let a genuine SOD carry an invented DG1's hash in its
unhashed tail.

## Public inputs

Five field elements, in this order, and the chain's params agree:

| Index | Value | Param |
| --- | --- | --- |
| 0 | `current_date` (YYMMDD) | `current_date_index` |
| 1 | registration binding | `address_index` |
| 2 | nullifier | `nullifier_index` |
| 3 | DSC commitment | `dsc_key_index` |
| 4 | identity commitment (`idc`) | `idc_index` |

The chain rejects any input of p or more before verifying, and pins
`current_date` to block time within the skew parameter.

## The registration binding

Input 1 is not an address. It is

    H(TAG_REG, Bytes(chain_id), idc, pc_anml, Bytes(ct_anml), pc_erth, Bytes(ct_erth), affiliate)

- `chain_id` first, so a registration seen on one network cannot be replayed
  onto another.
- `idc`, the commitment to the registrant's identity secret. It becomes a leaf
  of `x/personhood`'s identity tree.
- the two notes the first ANML and the registration reward are minted to, and
  their ciphertexts.
- `affiliate`: 0 for no referrer, else `H("earth.affiliate", handle)`. The
  chain resolves the handle when the registration executes and mints the
  referrer's half to the handle's address itself.

The chain recomputes the binding from the message, so a proof copied out of a
block cannot be redirected to other notes, another referrer or another chain.

## Identity commitment

The chain requires public input 4 to equal `MsgRegister.idc`, or refuses
with `ErrBadPublicInputs` (1103). This holds for a first registration, a
switch and a re-entry alike. Without it, a holder could register her passport
to an idc someone else chose, and the succession the chain writes would let
them receive her handle or Caretaker split by a move. With it, every identity
in a passport's chain is one whose secret its holder had when registering it,
so a handle or split cannot be sold to someone else's identity. The residual
is selling the passport itself: its DG1 and SOD with the current identity's
secret let a buyer register it to his own identity, move the handle or split
there and renew them while that identity is live, and re-register the passport
each year (a re-entry needs only passive-authentication data and his own
`id_secret`), until the passport expires or the seller switches it back.

An idc registers **once**. `x/personhood` keeps the set `UsedIdcs` (genesis
`used_idcs`) and refuses any idc in it in the ante, before the proof
(`ErrIdcUsed`, 1130); an idc whose registration is still live is refused
first, as a replay (`ErrRegistrationReplay`, 1123). Occupying someone's idc
needs its secret, so the set cannot be used to block anyone. The wallet
therefore derives a series of identity secrets from one recovery phrase,
`id_secret_g` for g = 0, 1, 2, …, and registers the lowest unused one each
time: a renewal after a lapse, or a fresh identity in the same wallet, needs no
new phrase. Notes, stake and the shielded address are the same for every
generation.

## Nullifier

Poseidon2 over DG1's issuing state (3 bytes), the 9-character document number
field, its check digit, the date of birth (6 bytes) and the optional-data
field, which carries the overflow of a number longer than nine characters
(`nullifier_preimage` in `circuits/poa_core`). Every byte is covered by the
SOD hash, so none of it can be chosen. It is the public dedup key only: every
later action uses nullifiers derived from the identity secret, never from
passport data.

The hash is unkeyed and fast, so the nullifier is recomputable by anyone who
holds those fields, and guessable where they are predictable: the check digit
follows from the number, many states issue numeric or near-sequential
numbers, the optional-data field is empty or a guessable personal number for
most, and the registration's country and DSC narrow the issue window. Given a
country and a date of birth, the number space is about 10^9 or less. A match
reveals the registration (time, country, DSC, referrer handle), not a wallet.

The nullifier is not renewal-stable. A nullifier built from something that
never changes, like the name, could be recomputed by anyone who knows a name
and a birthday. The cost is that a renewed passport, or a second passport,
gives a second registration until the first lapses.

## DSC commitment

ECDSA keys: `Poseidon2(curve tag ‖ x ‖ y)`, with tags P-256 1, P-384 2,
P-521 3, brainpoolP256r1 4, brainpoolP384r1 5, brainpoolP512r1 6, P-224 8 and
brainpoolP224r1 9. The tag separates same-width curves (P-256 from
brainpool256, P-384 from brainpool384). RSA keys: `Poseidon2(10 ‖ e ‖
modulus)`, the exponent being a circuit witness; tag 7 (RSA without its
exponent) is retired. `CURVE_TAG_*` in `poa_core` must equal `certs.Tag*` in
the chain for ever: append only, never renumber. The chain recomputes the commitment from the DSC certificate in the
transaction and requires it to equal public input 3, so the circuit cannot
sign with one key and name another.

## Trust store

`x/pki` holds the CSCA certificates, one record per certificate, seeded in
genesis from `csca/`. A registration carries its DSC certificate, which must
chain to a trusted, unrevoked CSCA and must not itself be revoked. A
certificate that is a CA, may sign certificates or is self-issued is refused
as a DSC. See the [trust store runbook](../operations/trust-store-runbook.md).

## Verifying keys

Genesis carries one key per passport circuit
(`networks/genesis/verifying-keys/`) and one per private circuit (action,
membership, move, stake, vote: `networks/genesis/shielded-verifying-keys/`). A key
change is a governance-approved upgrade whose handler swaps the keys at the
upgrade height, together with an app release that proves against them. Ship
the app before the height.

## Regenerating after a circuit change

1. `nargo compile --workspace` in `circuits/`, then copy `target/*.json` into
   the Android assets.
2. In the chain repo, update `tools/poafixtures` if the witness shape changed,
   then run `scripts/regen-poa-fixtures.sh ../earth-network-mobile`. It proves
   every variant and refreshes the test fixtures.
3. Write the new keys where the upgrade handler embeds them, base64 with no
   trailing newline, and test that the embedded keys equal the fixtures.
4. Ship the app build before the upgrade height.
