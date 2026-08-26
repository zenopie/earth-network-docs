---
sidebar_position: 1
title: Trust store runbook
---

# Trust store runbook

This page gives the procedure to revoke or add a passport certificate, and the
duration of each procedure.

`x/pki` controls which passports can register. A Country Signing Certificate
Authority (CSCA) signs the Document Signers (DSCs) of a country. A DSC signs
passports. The chain accepts a registration only if its DSC chains to a trusted
CSCA and is not revoked.

Both changes require governance. No person holds a key that can make them
directly. The authority is the gov module account, which has no private key.

---

## Emergency: a compromised Document Signer

**This procedure uses the expedited track.** The decision is made in advance.
The duration is one day, not seven.

Prepare all items below before you need them. A procedure that is drafted during
an incident extends the one-day path to several days.

### 1. Identify the key

The revocation applies to the **signing key** of the DSC, not to the
certificate. Revocation of the key covers each certificate that carries it. You
submit a *certificate*, and the chain derives the key from it. The authority
therefore never encodes a canonical public key manually.

```bash
# Confirm that the file parses and is the expected certificate
openssl x509 -in dsc.cer -inform DER -noout -subject -issuer -dates

# List registrations under it. x/pki has no DSC query; x/personhood has this one
earthd query personhood registrations-by-dsc <dsc-key> --node https://rpc.erth.network
```

### 2. Write the proposal

`revoke-dsc.json`:

```json
{
  "messages": [
    {
      "@type": "/earth.pki.v1.MsgRevokeDsc",
      "authority": "GOV_MODULE_ADDRESS",
      "certificate_der": "BASE64_DER_CERTIFICATE"
    }
  ],
  "metadata": "",
  "deposit": "5000000uerth",
  "title": "Revoke compromised DSC <country> <identifier>",
  "summary": "The reason. The date of discovery. The reporter. The known state of registrations already made with it.",
  "expedited": true
}
```

Get the authority address:

```bash
earthd query auth module-account gov --node https://rpc.erth.network
```

### 3. Submit

```bash
earthd tx gov submit-proposal revoke-dsc.json \
  --from <key> --chain-id earth-1 --gas auto --gas-adjustment 1.5
```

**The deposit is 5 ERTH and must be complete.** A proposal with a partial
deposit stays in the deposit period and the voting period does not start.

### 4. Vote and monitor

```bash
earthd tx gov vote <id> yes --from <key> --chain-id earth-1
earthd query gov proposal <id>
```

The proposal passes at 33.4% quorum and three quarters yes votes. The expedited
threshold is higher than the normal two thirds because the voting period is one
day instead of seven.

### 5. After the proposal passes

Revocation **is retroactive and starts automatically.** The passed proposal has
two effects. No further registration can use that signer. The registrations that
it already produced enter a queue for retirement. A second vote is not required.

The retirement is rate-limited. `purgeRevokedDscs` retires a limited batch at
each block, from the same budget as the expiry sweep, and takes its share first.
A signer with many registrations therefore drains over several blocks and not in
one block. Monitor completion:

```bash
# queued, then per batch, then complete
#   dsc_purge_started / registration_sweep_capped (reason=dsc_revoked) / dsc_purge_complete
earthd query personhood registrations-by-dsc <dsc-key> --node https://rpc.erth.network
```

ANML claims stop immediately, independent of the queue, because the claim path
re-reads the registration at each claim. The purpose of the purge is governance
weight. A stream stores its total weight and moves that weight only when a voter
is explicitly cleared. Weight that a revoked signer produced therefore stays
counted until its registrations are retired.

Publish which registrations were retired and the reason. In the normal case, the
holders of those registrations did nothing wrong. A compromised signer is a
failure of the issuing state. Those holders can register again when their
country has a signer that the chain trusts.

---

## Routine: adding a CSCA

This applies to a new country, or to a country that rotates its root. There is
no urgency. Use the normal 7-day track.

CSCAs come from the ICAO master list. Update the trust store instead of adding
the certificate manually, so that the repository and the chain agree:

```bash
# add the certificate, then regenerate
go run ./tools/pki-genesis csca/masterlist/allowlist.ml csca/additional/*.cer
```

For a chain that already runs, add the certificate with `MsgAddCsca`. Use the
same proposal structure as above, with `"expedited": false` and a 1 ERTH
deposit.

Keep `csca/` synchronised in both cases. If the repository and the chain hold
different sets of accepted passports, the repository is wrong, and the
difference stays undetected until a genesis export.

---

## Revoking a CSCA

This applies to the root of a country, not to one of its signers. Use it when
the state itself should no longer create identities that this chain accepts:
its root key is compromised, or governance decides not to trust it.

Understand the effect before you use this procedure. A CSCA in the trust store
can sign any number of Document Signers. Each of those can sign passports that
the chain counts as separate humans. This is the sybil surface and the reason
that the message exists. It is also the reason that a revocation excludes each
honest passport holder of that country.

**Track:** use expedited if the root is compromised. Use normal if the change is
a policy decision. Examine a policy decision that cannot wait seven days.

`revoke-csca.json`:

```json
{
  "messages": [
    {
      "@type": "/earth.pki.v1.MsgRevokeCsca",
      "authority": "GOV_MODULE_ADDRESS",
      "certificate_der": "BASE64_DER_CERTIFICATE"
    }
  ],
  "metadata": "",
  "deposit": "5000000uerth",
  "title": "Revoke CSCA <country>",
  "summary": "The reason. The known state of Document Signers issued under it, and the plan for registrations already made.",
  "expedited": true
}
```

Three properties of this message:

**One message covers the complete key.** A country holds several CSCA
certificates — renewals and link certificates — that share one signing key. Any
of them verifies a child signature. The chain revokes the *key* behind the
submitted certificate, so the sibling certificates are revoked with it. One
proposal for each certificate is not necessary. One proposal for each key is
necessary if the country has rotated to a new key.

**The effect is prospective only.** Existing registrations do not change and no
purge starts. Without further action, those registrations lapse within one
`registration_validity_seconds` period, which is one year at genesis. This
happens because a renewal re-verifies the Document Signer and that check now
fails. To retire them earlier, revoke the individual DSCs with `MsgRevokeDsc`
above, which does start a purge.

**Adding the certificate again reverses the revocation.** There is no un-revoke
message. `MsgAddCsca` with the same certificate is the reverse operation, and it
clears the revocation as a side effect. Know this before a proposal to re-add
the ICAO master list as routine maintenance restores a revoked root.

Then update `csca/` in the repository to match, as for an addition. A difference
between the trust store on disk and the trust store on chain causes a genesis
export to trust a certificate that nobody approved.

---

## Timing

| | Deposit | Voting period | Total |
| --- | --- | --- | --- |
| Expedited — revocation (DSC or CSCA) | 5 ERTH | 1 day | About 1 day |
| Normal — adding a CSCA | 1 ERTH | 7 days | About 7 days |

Quorum is 33.4% for both tracks. A normal proposal passes at two thirds yes
votes. An expedited proposal passes at three quarters. While the chain has one
validator, each vote passes. The delay is the voting period, not the result.

---

## Out of scope

**Retirement of the existing registrations of a country in one step.** A CSCA
revocation prevents new registrations. It does not remove existing ones. There
is no country-wide purge. This was considered and excluded: the operation
removes personhood from people who did nothing wrong, so it requires its own
decision and must not be a side effect of a root revocation. The available
methods are per-DSC revocation, or expiry at the end of the validity period.

---

## Before launch

- [ ] Confirm that the expedited track is correct for revocation, or record the
      reason that it is not.
- [ ] Replace `GOV_MODULE_ADDRESS` above with the actual value.
- [ ] Check the query commands against the released binary. They are written
      from the messages of the module, not from a live run.
- [ ] Decide who can submit a revocation, and how to contact that person outside
      working hours. At launch this is the genesis validator,
      `earth14e6sqtf5y7mtzwykqreewe9kg3w94t0f25d54a`. Its key is the
      `VALIDATOR_MNEMONIC` in the gitignored `.env` file. One person with one key
      is acceptable at launch. An unidentified person is not. A mnemonic in a
      `.env` file is not an acceptable location for this key after the chain
      carries value.
