---
sidebar_position: 4
title: DSC registry
---

# Passport PKI: on-chain DSC registry (Option C)

This is the design for a **trustless, permissionless DSC registry**. The chain
verifies the DSC-to-CSCA link **natively, in Go, one time for each DSC**. It
does not verify that link inside the ZK proof of each user. The registration
proof of the user therefore stays small: it covers SOD-to-DSC and membership
only, and it reuses the existing `lean_poa` circuit. RSA never runs in a proof
on a phone.

> **Status: implemented.** The complete stack is built and tested: `zk/poseidon2`
> (a Go Poseidon2 that matches the circuit), `x/pki/registry` (an append-only
> tree), `x/pki/certs` (native multi-curve certificate verification, 536 of 536
> actual CSCAs), the `x/pki` module (`MsgSubmitDSC` with `DscInclusion` and
> `Root` queries), and the binding of caretaker `MsgRegister` to the live root
> history. `tools/pki-genesis` seeds CSCAs from an ICAO master list. The mobile
> client (`chain/Pki.kt`) queries inclusion and submits DSCs. Remaining work:
> timing validation on a live devnet, and SOD-to-DSC circuits for each DSC
> algorithm for non-P256 passports. See section 12.

## 1. Trust model

The ICAO passive-authentication chain of trust:

```
CSCA (country root)  --signs-->  DSC (document signer)  --signs-->  SOD (passport)  --hashes-->  DG1 (MRZ)
     |                                  |                                |                            |
 governance-seeded              permissionlessly submitted,       proven in ZK by the         nullifier = Poseidon2(name‖DOB)
 masterlist (rare updates)      chain verifies DSC→CSCA natively   user (SOD→DSC + DG1)        (in-circuit)
```

Verification of each link:

| Link | Verified in | Cost | Frequency |
|---|---|---|---|
| CSCA is trusted | Governance masterlist, on-chain | — | Every 3 to 5 years |
| DSC-to-CSCA signature | **Native Go, on-chain, one time for each DSC** | About 1 ms | One time for each DSC, permissionless |
| SOD-to-DSC signature | **ZK proof (`lean_poa`), for each user** | About 13 s on the device | One time for each person |
| DG1, expiry, nullifier | ZK proof | — | For each person |

The expensive RSA-4096 operation runs **one time for each DSC, natively**. It
does not run in the circuit of each user and it does not require a trusted
curator. DSCs are public data: each passport SOD contains one. Submission of a
DSC on-chain therefore discloses nothing. The DG1 and identity of the user stay
private inside the ZK proof.

The result is a design that is fully trustless, because the chain verifies each
signature; that keeps the circuit small, because it reuses `lean_poa` with no
RSA inside the circuit; and that has no privileged curator. The only governance
action is maintenance of the small and stable CSCA masterlist.

## 2. Components

1. **CSCA masterlist** — the approximately 536 ICAO country roots, managed by
   governance. Seeded from `csca/masterlist/allowlist.ml`. It changes rarely.
2. **DSC registry** — an append-only Poseidon2 Merkle tree of validated DSC
   public keys. It grows through `MsgSubmitDSC`. `lean_poa` proves membership in
   its root.
3. **`MsgSubmitDSC`** — permissionless. The chain verifies DSC-to-CSCA natively
   and appends the DSC.
4. **`MsgRegister`** — the existing caretaker message that carries the `lean_poa`
   proof of the user. It now checks the `registry_root` of the proof against the
   **root history** of the DSC registry.

The proposed location is a dedicated `x/pki` module for the masterlist and the
DSC tree, with `x/caretaker` depending on it for the current roots. An
alternative is to place both in `x/caretaker`. A separate `x/pki` keeps the PKI
concerns separate.

## 3. State (`x/pki`)

```
Masterlist:      map[cscaKeyId]Csca                 // governance-managed
    Csca = { subject, subjectKeyId, notAfter, pubkeyDER }   // enough to verify a child sig

DscTree:         incremental Merkle tree (Poseidon2, BN254, depth D)
    nodes:       map[(level,index)] -> Field         // sparse; zero-subtree cache for empties
    nextIndex:   uint64
    root:        Field

RootHistory:     ring buffer (or set) of recent roots // for proof freshness (see §7)

DscByFingerprint: set[fingerprint]                    // dedup; fingerprint = sha256(SPKI)
DscLeaves:       map[index] -> DscLeaf                // so clients can fetch a path
    DscLeaf = { keyType, pubkey, leaf, notAfter, fingerprint }
```

Parameters: tree depth `D`, permitted key types and sizes, root-history length,
and the submission fee.

## 4. Messages

### `MsgSubmitDSC` (permissionless)

```
MsgSubmitDSC { submitter, dsc_cert_der }
```

The handler performs these steps:

1. `dsc = x509.ParseCertificate(dsc_cert_der)`.
2. Find the issuing CSCA in the masterlist. Match `dsc.AuthorityKeyId` to the
   CSCA `SubjectKeyId`. If that fails, match the issuer DN. If no CSCA matches,
   reject the message.
3. **Verify the DSC-to-CSCA signature natively:**
   `csca.CheckSignature(dsc.SignatureAlgorithm, dsc.RawTBSCertificate, dsc.Signature)`.
   Go handles RSA-PKCS#1v1.5, RSA-PSS, ECDSA, and the hash functions natively.
   Use the raw `CheckSignature` function. Do not use the full-chain `Verify`
   function, because ICAO certificates often violate strict X.509 policy. Use
   the AKI, SKI, and subject heuristics that already exist in
   `tools/epassport_verifier.py` in the backend.
4. Check that the certificate is current:
   `dsc.NotBefore <= blockTime <= dsc.NotAfter`.
5. Compute `fingerprint = sha256(dsc.RawSubjectPublicKeyInfo)`. If the
   fingerprint is already present, return without a change. The message is
   idempotent. Otherwise continue.
6. Compute `leaf = Poseidon2(canonicalPubkeyEncoding(dsc))`. The encoding
   depends on the key type and must match the circuit. See section 8.
7. Append `leaf` at `nextIndex`. Recompute `root` incrementally. Append `root` to
   `RootHistory`. Store `DscLeaves[nextIndex]`. Increment `nextIndex`.
8. Emit the event `{index, fingerprint, keyType, newRoot}`.

Native RSA and ECDSA verification takes microseconds. Charge a small fee or gas
amount to limit spam. The operation is idempotent and public, so the only effect
of abuse is growth of storage.

### Masterlist management (governance)

Use `MsgAddCsca` and `MsgRemoveCsca`, gated by governance. An alternative is a
genesis-seeded masterlist that governance updates in the style of
`MsgUpdateParams`. Seed it from `allowlist.ml`. See section 10.

## 5. Notes on native DSC-to-CSCA verification

- Go `crypto/x509`, `crypto/rsa`, and `crypto/ecdsa` cover each algorithm in the
  store. Measured content: the CSCAs are 374 RSA certificates, mostly RSA-4096
  and RSA-6144, and 1 EC P-521 certificate. DSCs add RSA-2048 and ECDSA P-256.
- ICAO certificates have known irregularities: NULL parameters in the
  AlgorithmIdentifier, non-positive serial numbers, and SKI or AKI mismatches.
  `tools/epassport_verifier.py` in the backend already handles these. Port its
  matching heuristics.
- RSA-PSS DSCs require `rsa.VerifyPSS` with the correct salt and MGF. RSA-PKCS
  requires `rsa.VerifyPKCS1v15`. `x509.CheckSignature` selects the function from
  the algorithm OID of the certificate.

## 6. Merkle registry and the one new dependency: Poseidon2 in Go

The circuit verifies membership with **Poseidon2 (BN254) at depth D**. The
on-chain tree must therefore compute identical hashes:

- `leaf = Poseidon2(pubkey field-bytes)`
- `node = Poseidon2([left, right])`

These must match `noir-lang/poseidon v0.3.0`, which is equal to
`@zkpassport/poseidon2` and is already validated against the circuit in
JavaScript. **First action item:** obtain a Go Poseidon2-BN254 implementation
that matches. Check the poseidon2 parameters in `gnark-crypto`. If they differ,
port the implementation and validate it with the vectors used for JavaScript.
For example, `Poseidon2(64 pubkey bytes)` must equal the leaf value of the
circuit. The remainder of Option C is standard Cosmos module work. Complete this
item first.

Use an incremental Merkle tree in the style of Semaphore or Tornado: store the
touched nodes and a zero-subtree cache for each level. Each append then costs
`O(D)` Poseidon2 hashes.

## 7. Change to user registration (caretaker `MsgRegister`)

At present, `verifyRegistrationProof` compares the `registry_root` of the proof
to the static value `params.DscRoot`. Change it to compare against the **root
history** of the DSC registry.

The reason for a history: the tree is append-only. A user builds a proof against
the root that existed when the user fetched the path of the DSC. By the time of
submission, a newer DSC can have changed the root. Membership in an older and
smaller append-only tree stays valid, so accept any root within a recent window:

```
if !pki.RootHistoryContains(proof.registry_root) { reject }
```

Keep a bounded ring buffer: the last N roots, or the roots within a period T.
Revoked DSCs are the only reason to prune the history. See section 9. The
remainder of `verifyRegistrationProof` does not change: the UltraHonk
verification, the `current_date` binding, and the nullifier extraction stay as
they are.

## 8. Circuit requirements

- **`lean_poa` does not change** for P-256 DSCs. The leaf is
  `Poseidon2(64-byte x‖y)`, membership is in the depth-D tree, and the root is a
  registry root. The circuit already performs SOD-to-DSC verification with
  ECDSA-P256, the hash bindings, the expiry check, and the name‖DOB nullifier.
- **The leaf encoding must match the chain for each key type.** For P-256, the
  leaf is `Poseidon2(64 bytes)`. For RSA, the leaf is `Poseidon2(modulus limbs
  or bytes)`. The `canonicalPubkeyEncoding` function of the chain and the leaf
  computation of the circuit must agree for each type.
- **One circuit is required for each DSC algorithm.** This is unavoidable and it
  is the minimum. The ZK proof must verify the SOD signature, and the algorithm
  of that signature is the algorithm of the DSC. The supported set is therefore
  P-256, which is complete, then RSA-2048, RSA-4096, P-384, and others. This set
  is **additive** and applies to the DSC side only. Option C needs **no
  CSCA-side circuits**, because the chain verifies that link in native Go. Each
  circuit is registered on-chain as its own `verifying_keys[algo]`, which the
  chain already supports.
- **Tree depth `D`.** `lean_poa` currently uses 16, which holds 65,536 DSCs.
  Consider `D = 20`, which holds approximately 1 million, for headroom. The cost
  is 4 additional Poseidon2 operations for each proof. Decide this before
  seeding.

## 9. Revocation (v1.1)

Append-only membership means that a revoked or compromised DSC stays valid
without further work. Two options: add a revocation set and prune the affected
roots from the history, or bind a `revocation_tree_root`, as zkpassport does,
and check non-revocation either in the circuit or natively. This is out of scope
for v1. It is recorded here so that it is not forgotten.

## 10. Genesis and seeding

- Extract the 536 CSCAs from `csca/masterlist/allowlist.ml` with
  `tools/extract_csca.extract_csca_ders`. Seed the masterlist at genesis, or
  through an initial governance batch.
- The DSC tree starts empty. It fills permissionlessly through `MsgSubmitDSC`.
  Optionally pre-seed known DSCs from the ICAO PKD or from collected
  certificates, so that early users do not each pay a submission fee.
- The off-chain `tools/registry-builder` is already written and validated
  against the circuit. It stays useful for test fixtures and to cross-check the
  on-chain tree root.

## 11. Client flow in the application

1. Scan the passport. Read DG1 and SOD. Extract the DSC.
2. Query `x/pki`:
   `DscInclusion(pubkey) → { root, index, pathBits[D], siblings[D] }`.
   - If the DSC is absent, send `MsgSubmitDSC(dsc_cert_der)`. Any account can do
     this and the chain verifies it natively. Then query again.
3. Build the `lean_poa` inputs with the existing `PassportInputs.leanInputs`,
   using the returned `root`, `pathBits`, and `siblings`. The application runs no
   Poseidon2 operations, because the chain supplies the path.
4. Prove on the device, which takes approximately 13 seconds. Then send
   `MsgRegister`.

## 12. Implementation phases

1. ✅ **Poseidon2-BN254 in Go** (`zk/poseidon2`), validated against the leaf and
   node hashes of the circuit. See section 6.
2. ✅ **`x/pki` module**: masterlist state and governance messages; the DSC
   incremental Merkle tree; `MsgSubmitDSC` with native certificate verification;
   the root history; and the inclusion query.
3. ✅ **Masterlist seeding** from `allowlist.ml`, with `certs.ParseMasterList`
   and `tools/pki-genesis`, which extracts 536 CSCAs.
4. ✅ **Caretaker integration**: `verifyRegistrationProof` checks that
   `registry_root` is in the pki root history. It uses the static
   `params.DscRoot` only when no pki keeper is wired.
5. ✅ **Client** in `earth-network-mobile`: the `chain/Pki.kt` inclusion query
   and `MsgSubmitDSC`; `PassportInputs.scannedDsc` and the scanner wiring.
6. ⏳ **Circuits for each DSC algorithm**: P-256 is complete in `lean_poa`. Add
   RSA-2048, RSA-4096, and the Brainpool SOD-to-DSC variants as coverage
   requires. These additions are additive and apply to the DSC side only.
7. ⏳ **Live devnet validation**: measure submission-to-inclusion timing against
   `ignite chain serve`.

## Open decisions

The selected default is in parentheses.

- Tree depth `D` (**16**, which matches the circuit; raise it with a coordinated
  recompile).
- `x/pki` as a separate module, or inside `x/caretaker` (**separate module**).
- Root-history size and policy (**64**, with ring-buffer pruning in
  `recordRoot`).
- Pre-seeded DSCs at genesis, or fully permissionless (**permissionless**;
  genesis seeds CSCAs only).
- Revocation design (**v1.1**: a revocation tree, or governance removal with a
  history prune).
