---
sidebar_position: 1
title: Launch checklist
---

# Launch checklist

This page lists the changes required before this repository produces an actual
chain release.

The word "release" covers two things here, and they fail in different ways. A
**software release** is a versioned `earthd` that any person can build, verify,
and run. A **network launch** is one genesis file that each node agrees on, with
a validator set that other operators can join. At present the repository does
the first partly and the second not at all. The container does not join a chain.
It *creates* one at first boot.

No item below is a defect in the protocol. The items are the difference between
a devnet that communicates only with itself and a network that other operators
can join.

---

## 0. Decisions required before genesis is frozen

These are decisions, not work items. Each later phase depends on them. None can
change after the file is signed and published.

- [ ] **The validator set at height 1.** Decided: launch with one validator, and
      state this publicly with the plan for expansion. The validator account is
      in genesis: `earth14e6sqtf5y7mtzwykqreewe9kg3w94t0f25d54a`, with 1,000
      ERTH. This is sufficient to self-delegate 100 ERTH and to fund governance
      deposits for several years. No other account has a pre-allocation. A
      prospective validator earns ERTH by registering a passport or by bidding in
      the liquidity auction. This is a better property than a founder allocation
      list.
      **Open:** the gentx itself, which requires the key and therefore occurs on
      launch day. Also the public statement.

      Note that there is no expansion plan to write, because nothing is closed.
      `max_validators` is 100, there is no allowlist, and a second validator
      requires only ERTH and `MsgCreateValidator`. The plan is the absence of a
      plan, and this is already published on docs.erth.network. The item to state
      at launch is the current fact of one validator. Do not state a roadmap to
      permissionless validation that already exists.

      State two consequences directly. A single validator can censor
      registrations, and unforgeable registration is the purpose of the chain. A
      single validator also collects the complete staking pillar of 1 ERTH/sec,
      which is 86,400 ERTH each day, for the period that it is alone. Neither is
      a defect. Both are damaging if a reader discovers them instead of reading
      them here.
- [ ] **The auction bid denom.** `networks/genesis.json` ships the liquidity
      auction as `AUCTION_STATUS_PENDING`, with `bid_denom: ""` and
      `end_time: 0`, holding 840,960 ERTH in each earmark.
      `MsgStartLiquidityAuction` is governance-only. Its own comment states that
      it is "only meaningful once IBC is live and the intended bid asset actually
      exists on the chain". The launch sequence is therefore: the chain starts,
      the IBC channel opens, and governance starts the auction. Record that
      sequence before genesis, because 1.68 million ERTH stays inactive on the
      dex module account until it occurs.
- [ ] **`min_deposit`.** `config.yml:56` still contains `# TODO: raise before
      mainnet`. A deposit of 1 ERTH to submit a proposal is not a mainnet value.
      This is the one governance value that you cannot copy from another chain.
      Set it to a meaningful fiat amount at the launch price of ERTH.
- [x] **A minimum gas price above zero.** The value is `0.005uerth`, in the
      entrypoint default and in the SDL. This is approximately 500 uerth for a
      typical transaction. It is per-node configuration and not a chain
      parameter, so a change requires a restart. A block gas limit is also set.
      `max_gas` was `-1`, so only the 22 MB block size limited a block, and the
      fee was the only control on a single transaction consuming all of it. The
      value is now 100M.
- [x] **The holder of the governance keys at launch.** No person holds them. The
      authority for each governance action is the gov module account, which has
      no private key. The actual question is who holds the voting power to pass a
      proposal. At launch this is the single validator, not because it holds
      anything special, but because it is the only staker. This resolves as
      other accounts stake.
      The decision that was required was timing, and it is recorded in
      `docs/TRUST_STORE_RUNBOOK.md`. A compromised Document Signer uses the
      expedited track, which is one day instead of seven, with the proposal
      drafted in advance.

---

## 1. Genesis must be an artifact, not an effect of booting

**This is the blocking item.** `docker/entrypoint.sh:29-62` copies
`networks/genesis.json`, rewrites `genesis_time` to the current time from
`date -u`, creates a validator key, and then runs `add-genesis-account`,
`gentx`, and `collect-gentxs`. Each node that starts this image therefore
computes a *different* genesis file and a different app hash, and cannot share a
network with any other node. The committed file confirms this: it contains one
account and `"gen_txs": []`.

- [ ] **Fix `genesis_time`** to a specific UTC instant in the future. Stop
      writing it at boot. The current behaviour exists for a valid reason:
      emission is prorated against elapsed time, so a stale genesis file pays the
      complete interval at height 2. `docker/README.md` records 125,485 ERTH
      minted in one block from a file that was one day old. The correct fix is a
      launch time close to the actual start of the chain, not a rewrite on each
      node.
      The POL retirement schedules behave the same way, for the same reason. A
      launch from the committed file today retires 2,996,952,143 LP shares in a
      single block, which is 4.4 days of the ten-year schedule. This is a
      measured value, not a prediction.
- [ ] **Collect the gentxs** into the file, or ship zero gentxs and have
      validators join with `MsgCreateValidator` after height 1. Phase 0 decides
      this.
- [x] **Set `consensus.version.app`.** The value is now `1`, in
      `networks/genesis/chain.json`.
- [x] **Remove the devnet accounts.** The ads-for-gas hot wallet is no longer in
      genesis. Its key was a devnet key that had been stored on a laptop, and a
      launch genesis file must not fund an operational wallet from such a key.
      **Consequence to handle after launch:** ads-for-gas holds no funds at
      height 1. Fund it from the validator, with a key that has never left a
      signer. Only the validator and the dex module now hold a balance.
- [x] **Publish the sha256 of the final genesis file.** The entrypoint part is
      complete: join is the default, a hash mismatch against
      `/etc/earth/genesis.json.sha256` is fatal, and the previous self-init path
      requires `DEV_INIT=1`. `docker/entrypoint_test.sh` covers all three paths
      with 16 assertions, verified by mutating the entrypoint in three ways.
      **Open:** placing the hash in the release notes, which requires a release
      to exist.
- [x] **Stop using `--keyring-backend test` on the init path.** The join path
      creates no keys, so the test keyring and `VALIDATOR_MNEMONIC` now exist
      only under `DEV_INIT=1`, where the chain is disposable by design. The
      consensus key of an actual validator goes behind `PRIV_VALIDATOR_LADDR`.
- [x] **Make genesis generation reproducible.** `scripts/build-genesis.sh` builds
      `networks/genesis.json` from `networks/genesis/`, described in its README,
      and writes a `.sha256` file beside it. The output is sorted canonically, so
      two operators who run it produce identical bytes. `make genesis-check`
      fails if the artifact differs from its sources. `go test ./deploy/...`
      asserts that `bank.supply` equals the sum of balances; that no account
      outside `accounts.json` holds a balance; that the balance of the dex module
      equals the reserve of pool 1 plus both auction earmarks; that the LP supply
      is `sqrt(reserve_erth * reserve_token)`; that the retirement schedule is
      sized to exactly that value; that each verifying key and the CSCA trust
      store are seeded; and that `config.yml` has not diverged from the launch
      sources again. It had diverged: the pre-mine split in `6dd49f3` never
      reached it.
      **Open here:** `genesis_time` and the gentxs, both above.

---

## 2. There must be a method to join

- [x] **Expose p2p.** The `Dockerfile` already exposed 26656. An earlier version
      of this entry stated otherwise and was incorrect. The actual gaps were
      around it: `docker-compose.yaml` published only 1317 and 26657, the Akash
      SDL exposed no p2p port, and the entrypoint set no `external_address`. A
      containerised node therefore advertised its pod address to each peer and
      could never be dialled.
      Current state: compose publishes 26656; the SDL exposes it with
      `global: true` on a provider port instead of through the tunnel, because
      peers use raw TCP and cloudflared serves HTTP; and the entrypoint accepts
      `EXTERNAL_ADDRESS`, `SEEDS`, and `PERSISTENT_PEERS`, applied at each start,
      with a log message when the external address is unset.
      **Open on Akash:** adding the port is a structural SDL change, so it
      requires a lease replacement, and closing the lease destroys the volume:
      genesis, the consensus key, and each account. Perform it during the genesis
      cutover, while no state is worth keeping. The provider assigns the external
      port, so read it from the lease and set `EXTERNAL_ADDRESS` to the host of
      the provider and that port.
- [ ] **Publish seeds.** At minimum one seed node as `nodeid@host:26656`, in the
      README and in the release notes.
- [x] **Disable `--api.enabled-unsafe-cors`** for validator nodes. The setting is
      now `API_UNSAFE_CORS`, disabled by default, and it logs a warning when
      enabled.

      This was measured against the live devnet instead of assumed, which changed
      the conclusion two times. The LCD returns `access-control-allow-origin: *`
      to any origin, from the flag on the node. The RPC returns **no CORS headers
      at all**: not from the node, where `cors_allowed_origins = []`, and not
      from Cloudflare, which adds none. A browser has therefore never been able
      to reach the RPC cross-origin on any deployment of this chain, and the
      page-level traffic of the web application must use the LCD.

      Keplr conceals part of this. Signing occurs in the extension, and
      `keplr.sendTx` broadcasts from its background context. Neither is subject
      to page CORS. Any operation that the page performs itself, such as a
      `StargateClient` query or a `SigningStargateClient` broadcast, is subject
      to it.

      The new `RPC_CORS_ORIGINS` accepts a comma-separated allowlist and writes
      `cors_allowed_origins` into `config.toml`. It is applied at each start, so
      an origin can change without deleting the volume. The SDL sets it to
      `https://erth.network`. The LCD stays all-or-nothing, because that is the
      only option that the SDK offers: `server/config` has a boolean and no
      allowlist.
- [ ] **Separate the browser-facing LCD from the block producer.** One node is
      currently both, so `API_UNSAFE_CORS=1`, which is literally `*`, runs on a
      validator. There are two solutions and they combine: run a separate
      read-only LCD and RPC service, and set the header at Cloudflare, which
      already terminates both hostnames and can rewrite the header for each
      origin.
- [x] **Set snapshot and pruning defaults.** `SNAPSHOT_INTERVAL` is 1000, which
      is approximately 80 minutes at 5-second blocks, and `SNAPSHOT_KEEP_RECENT`
      is 5. Both are in the entrypoint and are applied at each start, so a
      restart changes the cadence. The SDK default is 0, which disables
      snapshots. With that default, no node offers snapshots and no operator can
      state-sync. A node cannot produce a snapshot for a height that it has
      passed, so a launch with that default cannot be corrected later.
      Verified on a live chain: five snapshots on disk, with `creating` and
      `completed` in the log.
      Pruning profiles for each node role, and the client-side state-sync
      procedure, are in `docs/JOIN.md`. This chain gains more from state sync
      than most chains, because a replay re-executes the zkSNARK verification of
      each registration. Sync cost therefore increases with adoption as well as
      with time.
- [x] **Write the join procedure.** This is `docs/JOIN.md`, executed completely
      against the released binary. Two values wait for launch day: the seed
      address and the genesis hash.
- [x] **Review the slashing window.** Both values were SDK defaults and not
      decisions. A 100-block window at 50% jails a validator after 50 missed
      blocks, which is about four minutes at 5-second blocks. With one validator,
      a container restart therefore halts the chain and slashes 1% of its stake.
      The values are now 10,000 and 0.05, which are the settings of the Hub, and
      which give approximately 13 hours. With a single validator, the usual cost
      of leniency — an inactive validator that stays in the set — does not apply.
      `historical_entries: 10000` is exactly the SDK default
      (`DefaultHistoricalEntries`), not 100 times it. The earlier note was
      incorrect.

---

## 3. Binary release engineering

- [x] **Publish binaries.** `release.yml` now runs on a version tag and builds
      `earthd` natively on Linux amd64 and arm64. Native builds are required
      because the Barretenberg verifier uses cgo, so the cross-compilation in
      Ignite could never have worked. The workflow publishes tarballs, a
      `checksums.txt` that also covers `networks/genesis.json`, and the genesis
      file, with the genesis sha256 in the release notes.
- [x] **Build the image with the version ldflags.** `VERSION` and `COMMIT` are
      build arguments, passed from the tag and from `github.sha`. The build uses
      `-trimpath`, so the binary does not vary with the checkout path. The
      toolchain is pinned to `golang:1.25.10-trixie` instead of floating on
      `1.25`.
- [x] **Stop pushing `:latest`.** The image is built and pushed under the version
      tag only, and the digest is read back from that tag.
- [ ] **Reproducible builds.** cgo and a prebuilt Aztec archive mean that two
      operators can produce verifiers that behave differently. That is a
      consensus fault, not a packaging inconvenience. Add `-trimpath`, pin the
      toolchain, publish a sha256 for each platform, and make `verifier-libs.yml`
      the only source of `libbarretenberg.a`, verified against
      `third_party/barretenberg-go/checksums.json` at build time.
- [x] **Remove compiled artifacts from the tree.** `poafixtures` is removed. It
      was a built binary that nothing referenced, committed by accident, and it
      is regenerable with `go run ./tools/poafixtures`. `.gitignore` now covers
      it and stray `earthd` builds.
      **`lib/darwin_arm64/libbarretenberg.a` stays, deliberately.** It is the
      development platform. Removal would mean that a new clone cannot build
      until it downloads approximately 100 MB from Aztec. The blob is also
      already in the history, so removal would not reduce the size of a clone:
      `.git` is 35 MB in total, and object code compresses well. The library of
      each other platform is now ignored, so a local
      `build-wrapper.sh --platform linux_amd64` cannot add another 50 MB by
      accident.
- [x] **Add a `LICENSE`.** Apache 2.0, selected over MIT for the patent grant and
      because the Cosmos SDK uses it.
- [x] **Add a `CHANGELOG.md`.** It is written for operators and not as a summary
      of commits. Consensus-affecting changes are listed separately, because for
      a chain those changes are breaking regardless of the size of the diff. The
      release workflow links to it.

---

## 4. The upgrade path must be exercised one time

`app/upgrades.go` is well built. It has named handlers, a store loader, and
skip-height handling. But `var Upgrades = []Upgrade{}` is empty, so none of it
has run. An untested upgrade path is discovered during an upgrade.

- [x] **Rehearse a no-op upgrade completely.** `scripts/rehearse-upgrade.sh` does
      this on a temporary chain. It passes an actual proposal, waits for the
      halt, confirms that the old binary *refuses* to continue, rebuilds with a
      matching handler, and confirms that the chain continues. All three
      behaviours are correct.
      The rehearsal found the error that costs a week: the plan height must be
      after the **end of the voting period**, not after the current height.
      `x/upgrade` rejects a plan whose height has passed at the time of
      execution, and the proposal then reports `PROPOSAL_STATUS_FAILED`: it
      passed the vote and failed to apply. With a 7-day voting period, the
      required margin is approximately 120,000 blocks. The first run of the
      script failed in this way.
- [x] **Cosmovisor: decided against bundling.** Upgrades are a binary change at a
      halt height, and `docs/UPGRADES.md` is written from a rehearsal of that
      procedure. An operator who wants cosmovisor can run it around the released
      binary. Nothing here prevents that.
- [x] **Document the halt-height procedure.** This is `docs/UPGRADES.md`, written
      from the rehearsal and not from the SDK documentation. It covers the
      plan-height arithmetic, `StoreUpgrades` when the module set changes,
      `--unsafe-skip-upgrades` and the reason that it must be coordinated, and
      the meaning of a node that halts again.
- [ ] **Decide the store-migration policy** for the modules that are most likely
      to change shape: the `x/personhood` parameters, which are the verifying keys
      and the signal indices, and the `x/dex` auction state.

---

## 5. Open correctness gates

- [ ] **The simulation operations are stubs.** Each file in
      `x/dex/simulation/*.go` and `x/personhood/simulation/*.go` contains a
      `// TODO: Handle the X simulation`, so `app/sim_test.go` exercises the SDK
      and no part of this chain. These are the least expensive confidence
      available for the invariants of the AMM and the personhood state machine,
      and once written they run in CI permanently.
- [x] **No invariants were registered.** This is complete for `x/dex`, which
      holds the pre-mine. `x/dex/keeper/invariants.go` checks at each block, in
      the EndBlocker, that the records of the module and its bank balance agree
      **exactly**. It detects both a shortfall, which is a withdrawal that will
      not be payable, and a surplus, which is coins that should have been burned
      and were not. Nothing else would detect a surplus. A breach returns an
      error from EndBlock, which halts the node. That is the intended result: an
      upgrade can recover a halt, and nothing can recover a silent drain of the
      pre-mine. A second check enforces the LP-reward denominator that
      `lp_rewards.go` had only asserted in prose. Share backing — escrowed
      withdrawals plus the position of the protocol, against the shares that
      exist — walks the unbonding queue, so it runs in tests instead of at each
      block.
      Verified by mutation. Five separately injected defects are each detected
      with an exact figure: a retirement that reduces a reserve without burning,
      a swap fee that is deducted but left in the pool, an unbonding that is paid
      without reducing the reserve, a reward credited two times, and
      double-counted escrow. A live node ran 62 blocks from
      `networks/genesis.json` with no breach.
      **Not complete:** the reward index of `x/allocation` and the ANML
      accounting of `x/personhood` have no equivalent check.
      **Note:** `x/crisis` is deprecated in SDK v0.53 and is removed in the next
      release, so the checks are enforced directly instead of registered through
      it. This also required blocking the module accounts of the chain from
      receiving ordinary transfers, in `app/app_config.go`. Without that, any
      account could halt the chain with a `MsgSend`, and the check would have to
      weaken to "holds at least what it owes", which stops detecting the second
      class of defect completely.
- [x] **The `x/allocation` invariant walked every option at every block.** Fixed.
      `AssertInvariants` ran in the EndBlocker, and `CheckStreamWeight` summed a
      stream by walking each of its options and decoding each record completely.
      Adding an ADDRESS option is permissionless, so that walk grew for a
      one-time fee: one ERTH burned, gas paid in the block that stored the row,
      and each node re-reading it at every later block. This is the same shape
      that `x/dex` had before `TestPoolSetIsNoLongerUnbounded`.
      The sum is now maintained instead of recomputed. `setOption` is the only
      writer of an option record. It moves a per-stream `summed_weight` by the
      actual change in the balance of each option, and the EndBlocker compares
      that against `total_weight`. This is two numbers for each stream, at any
      option count. `TestInvariantCostIsFlatInOptionCount` measures 4,246 gas at
      five options and the same 4,246 gas at five hundred.
      This stays an actual check, and not a number that agrees with itself,
      because the two aggregates are maintained at different sites and from
      different quantities. `resyncVoter` moves the total by the complete weight
      of a voter. The sum moves for each option and is never clamped. The clamped
      case is the case that proves the check, and it still halts.
      The bounded pair does not detect an option written outside `setOption` with
      the total unchanged: both aggregates miss it and therefore agree. That is a
      defect in the module and not an action that a user can cause. The walk that
      detects it is now in `AssertInvariants`, which the tests run after each
      operation and which an operator can run against a halted node to determine
      which of the two numbers is incorrect. Genesis validation walks the options
      at each import, and the running sum is rebuilt from them at that point.
- [x] **Permissionless options grew the state and the query with no reverse
      operation.** Fixed in the two places where it appeared. Adding an ADDRESS
      option is permissionless and its fee is paid one time, so each option ever
      added was a row stored permanently, whether or not any voter pointed at it.
      The `Options` query returned all of them on a route that costs the caller
      nothing.
      Options now expire. Thirty days with no weight removes the option, together
      with any rewards that it earned and that nobody claimed. Nothing is burned:
      the accrued ERTH of an option is minted at the time of a claim, so a
      forfeited balance is issuance that does not occur. Any account may trigger
      a claim on an ADDRESS option, and the payout goes to the recipient
      regardless of the sender. The INTEGRATED options of governance are never
      swept. The schedule is a queue ordered by due date, maintained at each
      option write, so a quiet block reads one key and a busy block removes at
      most 20. The cap is in the per-block budget in `app/block_budget_test.go`.
      The query is paged at 100 entries. Walking the complete table is still
      possible, but page by page, as separate requests that a node can meter.
- [x] **The public-input schema is fixed and the values are correct.** This entry
      previously stated that `verifyRegistrationProof` carried placeholder
      indices that could not be adjusted later without invalidating each
      registration. Both statements are out of date, and the entry was repeated
      as a launch blocker long after the work was complete.
      The positions are governance parameters: `params.NullifierIndex`,
      `params.DscKeyIndex`, and `params.CurrentDateIndex`, seeded 1, 2, and 0.
      They match the circuits. `main.nr` takes one public input, `current_date`,
      and returns `(nullifier, dsc_key)`, which gives
      `[current_date, nullifier, dsc_key]`. All seven signature variants share
      that public surface, so one set of indices serves each algorithm.
      Checked against an actual proof instead of by reading the source:
      `x/personhood/keeper/testdata/lean_poa/public_inputs` holds exactly three
      field elements — `250101`, then `expected_nullifier`, then
      `expected_dsc_key` — and the handset divides three values in the same
      order.
- [x] **The verifying keys are seeded and `dsc_root` no longer exists.** Seven
      distinct keys ship in `networks/genesis/verifying-keys/`, one for each
      algorithm. The `lean_poa` key is byte-identical to the key that the keeper
      test uses to verify an actual bb proof, so these are `bb write_vk` output
      from the actual circuits and not demonstration keys. `dsc_root` is
      `reserved` in the params proto. A registration now carries the Document
      Signer certificate, checked against the 539 CSCAs that genesis seeds from
      `csca/`.
- [x] **CI builds the Linux verifier libraries; they are not checked in.**
      `third_party/barretenberg-go/lib/` holds `darwin_arm64` only. This appears
      to be a gap and is not one. `release.yml` builds `libbarretenberg.a` on a
      native runner for each architecture, without cross-compilation, because
      earthd links the verifier through cgo. It does this before it builds
      `earthd`, and it verifies the checksum against `checksums.json` at bb
      v5.0.0. `verifier-libs.yml` builds the same libraries at release time and
      attaches them. The Dockerfile builds the library in the image. The
      checked-in macOS library is a development convenience. The practice being
      avoided is committing multi-megabyte archives for each platform.
- [ ] **Prove on actual hardware, end to end.** The on-device path is written:
      `PassportProver` calls `NoirProver` for a bb v5.0.0 poseidon2 proof of
      `lean_poa`, divided into the `(proof, public_signals)` form of the chain.
      But no actual passport has been scanned on an actual handset into an actual
      node. Keep the bb version of noir_android equal to the v5.0.0 version of
      the chain, or the chain rejects large-circuit proofs on arrival.
- [x] **Write the trust-store runbook.** This is `docs/TRUST_STORE_RUNBOOK.md`.
      Revocation uses the expedited track. Both gaps that it previously named are
      closed. `MsgRevokeCsca` gives the root of a country a one-day path, and it
      revokes the signing key instead of the certificate, so it also covers a
      CSCA that issues certificates that it should not issue. The remaining gap
      is deliberate: revoking a CSCA is prospective, and there is no country-wide
      purge of registrations already made.
- [ ] **External audit** of `x/dex`, covering auction settlement and LP
      accounting; `x/allocation`, covering the reward index that two streams
      share; and the verifier shim in `zk/ultrahonk`. A defect in these three
      places is unrecoverable rather than embarrassing.

- [x] **Genesis export dropped most of the state of the chain.** `GenesisState`
      contained parameters only for `x/allocation`, `x/personhood`, and `x/pki`.
      An `earthd export` followed by an import therefore discarded each
      registration, and with them the nullifier set, which reset the anti-Sybil
      property to a state where nobody had registered. It also discarded each
      revoked Document Signer and each allocation option, vote, and reward index.
      Fixed: all three modules now carry their state. Derived indexes are rebuilt
      at InitGenesis instead of exported, and `Validate` rejects a malformed file
      at import instead of halting the chain at height 1.
      `TestAppImportExport` passed throughout, because the simulation operations
      are stubs, so the simulation never created a registration, an option, or a
      revocation. Empty state round-trips correctly. This is a concrete argument
      for the simulation operations above.
- [x] **`x/pki` stored one certificate for each signing key, not one for each
      certificate.** `networks/genesis.json` carries 539 CSCAs and the chain held
      366. `Cscas` was keyed by `cscaID`, which is the SKI, so certificates that
      share a key overwrote each other. 173 certificate bodies were dropped at
      InitGenesis and never reached the chain.

      This was measured first, because the measurement determined the fix. Of 536
      parsed certificates there are 366 distinct SKIs, and 337 certificates share
      an SKI with another certificate. **All 337 also share a public key.** None
      is an actual collision. They are renewals and link certificates for one
      signing identity, and any of them verifies a given signature. Exactly one
      SKI group spans more than one subject DN.

      Keying issuer *lookup* by SKI was therefore correct, because the AKI of a
      DSC is the SKI of its issuer. The defect was that one map served as both
      the lookup index, which is per key, and the record store, which is per
      certificate. These are now separate:

          Cscas      certID (sha256 of DER) -> Csca
          CscaBySKI  (SKI, certID)
          CscaByDN   (sha256(DN), certID)

      `issuerCandidates` walks `CscaBySKI` under the AKI of the DSC instead of
      performing one `Get`. It therefore returns each certificate that carries
      the key, and not an arbitrary sibling. The certificates share a key but
      differ in validity period, and the caller can select the certificate that
      was valid only if it receives all of them. A live chain now stores and
      exports all 539.

      No migration is required. The store layout changed, but the chain has not
      launched, because `app/upgrades.go` is empty, and `networks/genesis.json`
      is byte-identical, because the file always carried 539. The import
      collapsed them. A devnet with state worth keeping requires a restart from
      genesis.

---

## 6. Documentation that the launch produces

- [x] `docs/JOIN.md` — written, and executed completely against the released
      binary. That execution found a missing step: the node refuses to start
      until `minimum-gas-prices` is set, with an error that does not name the
      file. Three values stay `TBD` until launch: the seed address, the genesis
      hash, and the gas price.
- [ ] Release notes for each tag: binary checksums; the genesis sha256, for the
      launch tag only; and the upgrade name and height, for upgrade tags only.
- [x] **A documentation site.** `docs-site/` uses Docusaurus, which the Cosmos
      SDK and most Cosmos chains use. `.github/workflows/docs.yml` publishes it
      to `docs.erth.network`. It has six pages, covering the description of
      Earth, registering, emission, using the application, governance, and
      running a node. Each page has an edit link to its markdown source, and the
      build fails on a broken link. Operational guides stay in `docs/` next to
      the code and are linked instead of copied, so they cannot diverge within a
      release.
      **Two one-time setup steps remain, both outside this repository:** a
      `CNAME` DNS record for `docs` that points at `zenopie.github.io`, with
      Cloudflare SSL mode **Full** and not Flexible, because Flexible loops
      against Pages; and GitHub Settings → Pages with the source set to *GitHub
      Actions* and the custom domain set.
- [x] Replace the Ignite boilerplate in the readme. Removed: the `git tag v0.1`
      instructions for a workflow that no longer operates that way, the
      `get.ignite.com` install line that pointed at the wrong repository, and the
      Vue scaffolding section. The replacement covers running a node,
      development, releasing, and an index of the documentation.

---

## Items that are already correct

These are recorded so that no person changes them during the launch work:

- The SDL pins the image by **digest**, and CI rewrites it at release, in
  `deploy/akash/deploy.yaml:22-28`. This is correct, and for the stated reason.
- Genesis state is on a mounted volume, and the entrypoint distinguishes a first
  boot from a restart by the presence of `genesis.json`. The failure mode that
  this avoids — a redeploy that silently starts a different chain — is the
  correct one to protect against.
- The remote consensus signer path exists and fails closed. See
  `deploy/akash/REMOTE_SIGNER.md`. This is the mechanism that the key of an
  actual validator requires. It is not yet enabled.
- Governance parameters are already set to Cosmos conventions, with the reasoning
  recorded. The invariant
  `max_deposit_period + voting_period = unbonding_period` in `config.yml` is a
  detail that chains often get wrong.
- `community_tax: 0` and the inactive `x/mint` parameters are deliberate and
  documented, so genesis reports accurately instead of reporting inflation that
  the chain does not produce.

## Explicitly not blocking

- The IBC relayer that ships in the same image and stays inactive until the SDL
  enables it.
- `x/dex` simulation coverage beyond the operations listed above.
- The `ignite chain serve` developer flow, which should continue to work through
  `DEV_INIT=1` and `config.yml`.
