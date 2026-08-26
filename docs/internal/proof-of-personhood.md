---
sidebar_position: 2
title: Proof of personhood
---

# Proof of personhood: status and remaining work

The selected direction is **Noir and Barretenberg UltraHonk, proved on the
handset**. It was selected over Self for two reasons. The sensitive step stays
on the device, where Self moves it to a TEE. It also requires no trusted setup
for each circuit. Registration proofs are bb v5.0.0 UltraHonk proofs, in the
poseidon2 flavor, verified natively on-chain.

The circuits are **ours**, not the circuits of zkPassport. They are in
`circuits/` in the mobile repository: `poa_core` and seven `lean_poa` variants,
one for each Document Signer signature algorithm. The items taken from
zkPassport are the approach and the proving library (`noir_android`). The
circuit is not taken from zkPassport.

## Complete

- [x] **On-chain verifier** — `zk/ultrahonk`. These are CGo bindings to the
      Barretenberg verifier (`third_party/barretenberg-go`, with the native
      library rebuilt against bb **v5.0.0**). It verifies actual bb 5.0.0
      UltraHonk proofs and it is sound: it rejects modified proofs. It uses the
      poseidon2 flavor. See `third_party/barretenberg-go/README.md`.
- [x] **Caretaker integration** —
      `MsgRegister{proof, public_signals, signature_algorithm}` calls
      `verifyRegistrationProof`. That function selects the bb verifying key from
      `params.verifying_keys`, calls `ultrahonk.Verify`, checks the `dsc_root`
      binding, and returns the nullifier. The keeper test passes against an
      actual proof.
- [x] **Build infrastructure** —
      `third_party/barretenberg-go/scripts/build-wrapper.sh` and
      `checksums.json`, pinned to v5.0.0 for each platform.

## Remaining

1. **On-device proving, complete, on actual hardware.** The code exists.
   `PassportProver` calls `NoirProver` for a bb v5.0.0 poseidon2 UltraHonk proof
   of the `lean_poa` circuit, then divides it into the `(proof, public_signals)`
   form that the chain requires. The untested part is an actual passport,
   scanned on an actual handset, registering against an actual node. Observe the
   version requirement in that file: the bb version in noir_android must match
   the v5.0.0 library of the chain, or the chain rejects large-circuit proofs.

   This is the only open item. The validator native libraries are not an open
   item. `lib/` contains `darwin_arm64` for development. `release.yml` builds
   the Linux libraries for each architecture on native runners,
   `verifier-libs.yml` attaches them to releases, and the Dockerfile builds them
   in the image.

## Settled — do not reopen

- **The public-input schema is fixed and correct.** The positions are governance
  parameters. `x/personhood/keeper/registration.go` reads them as
  `params.NullifierIndex`, `params.DscKeyIndex`, and `params.CurrentDateIndex`.
  Genesis sets them to **1, 2, and 0**.

  These values match the circuits. `main.nr` takes one public input,
  `current_date`, and returns `(nullifier, dsc_key)`. The public array from
  Barretenberg is therefore `[current_date, nullifier, dsc_key]`. All seven
  variants — P-256, P-384, the three brainpool curves, RSA-2048, and RSA-4096 —
  have an identical public surface. One set of indices therefore serves each
  algorithm.

  This was confirmed against an actual proof, not by reading the source.
  `x/personhood/keeper/testdata/lean_poa/public_inputs` contains exactly three
  field elements: `250101` (a YYMMDD date), then `expected_nullifier`, then
  `expected_dsc_key`. The phone uses the same layout:
  `PassportProver.NUM_PUBLIC_INPUTS = 3`, divided in that order.

  An earlier version of this file stated that these were placeholders, and that
  an error in them would invalidate each registration. Both statements are now
  incorrect. The claim remained in the file after the fix and was repeated as a
  launch blocker.

- **The verifying keys are seeded and are the actual keys.** Seven distinct keys
  are in `networks/genesis/verifying-keys/`, one for each algorithm. The
  `lean_poa` key is byte-identical to the key that the keeper test uses to
  verify an actual bb proof. These are therefore `bb write_vk` output from the
  actual circuits, not demonstration keys.

- **`dsc_root` is removed.** It pinned a Merkle root of the certificate registry
  when that registry was off-chain. The field is now `reserved` in
  `proto/earth/personhood/v1/params.proto`. A registration now carries the
  Document Signer certificate itself, checked against the CSCA trust store. That
  store holds 539 certificates, seeded in genesis from `csca/`.
