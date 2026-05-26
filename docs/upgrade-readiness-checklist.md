# Protocol Upgrade-Readiness Checklist

End-to-end checklist for bumping `CONTRACT_VERSION` in `contracts/stream/src/lib.rs`, rotating the WASM on Stellar, and safely cutting over in-flight streams.

**Source of truth:** `contracts/stream/src/lib.rs` (`CONTRACT_VERSION` constant)  
**Cross-references:** [docs/DEPLOYMENT.md](./DEPLOYMENT.md) · [docs/maintainer-security-checklist.md](./maintainer-security-checklist.md) · [docs/upgrade.md](./upgrade.md)

---

## When to bump CONTRACT_VERSION

Bump `CONTRACT_VERSION` before deploying any of the following:

| Change type | Bump required? |
|---|---|
| Remove or rename a public entry-point | **Yes** |
| Change parameter type or order on any entry-point | **Yes** |
| Change a `ContractError` discriminant value | **Yes** |
| Change emitted event topic or payload shape | **Yes** |
| Change persistent storage key layout | **Yes** |
| Add a new entry-point (purely additive) | Recommended |
| Internal refactor — identical external behaviour | No |
| Documentation-only change | No |
| Gas optimisation — identical observable behaviour | No |

See [docs/upgrade.md](./upgrade.md) for the full policy and version history.

---

## Phase 1 — Pre-upgrade preparation

### 1.1 Code changes

- [ ] **Bump `CONTRACT_VERSION`** in `contracts/stream/src/lib.rs`.  
  Update the doc comment above the constant to describe what changed.
- [ ] **Update `docs/upgrade.md`** — add a row to the version history table.
- [ ] **Update `docs/streaming.md`** and any other docs that reference the changed behaviour.
- [ ] **Update `docs/audit.md`** if entry-point signatures or invariants changed.

### 1.2 Test suite

- [ ] Run the full test suite and confirm it passes:
  ```bash
  cargo test -p fluxora_stream
  ```
- [ ] If entry-point signatures changed, update all callers in:
  - `contracts/stream/tests/integration_suite.rs`
  - `contracts/stream/src/test.rs`
  - Any snapshot tests in `contracts/stream/tests/event_snapshots_suite.rs`
- [ ] Run property tests:
  ```bash
  cargo test -p fluxora_stream -- --include-ignored proptest
  ```
- [ ] Confirm no regressions in adversarial auth tests:
  ```bash
  cargo test -p fluxora_stream --test adversarial_auth
  ```

### 1.3 WASM build and checksum

- [ ] Build the release WASM:
  ```bash
  cargo build --release -p fluxora_stream --target wasm32-unknown-unknown
  ```
- [ ] Update the WASM checksum file:
  ```bash
  bash script/update-wasm-checksums.sh
  git add wasm/checksums.sha256
  git commit -m "chore: update wasm checksums for v<NEW_VERSION>"
  ```
- [ ] Verify the checksum locally:
  ```bash
  bash script/verify-wasm-checksum.sh
  ```

### 1.4 Data-key migration verification

- [ ] List all `DataKey` variants in `contracts/stream/src/lib.rs` and confirm:
  - No existing discriminant values were changed.
  - New keys use a discriminant not previously used.
  - Removed keys are documented (old entries will remain in storage but become unreachable).
- [ ] If storage layout changed, document the migration path in `docs/upgrade.md`.

### 1.5 Announce the upgrade

- [ ] Notify all known integrators (wallets, indexers, treasury tooling) with:
  - The new `CONTRACT_VERSION` value.
  - The expected new `CONTRACT_ID` (Soroban contracts are not upgradeable in-place).
  - The cutover date and the date the old contract will be abandoned.
  - Instructions for recipients to withdraw accrued funds from the old instance.
- [ ] Allow a minimum **7-day notice window** before cutover for active streams.

---

## Phase 2 — Testnet deployment and verification

### 2.1 Deploy new contract to testnet

```bash
# Build release WASM
cargo build --release -p fluxora_stream --target wasm32-unknown-unknown

# Upload WASM to testnet
stellar contract upload \
  --wasm target/wasm32-unknown-unknown/release/fluxora_stream.wasm \
  --source <DEPLOYER_SECRET_KEY> \
  --network testnet

# Deploy contract instance
stellar contract deploy \
  --wasm-hash <WASM_HASH_FROM_UPLOAD> \
  --source <DEPLOYER_SECRET_KEY> \
  --network testnet
# Save the output CONTRACT_ID
```

### 2.2 Initialize and verify

```bash
# Initialize the new contract
stellar contract invoke \
  --id <NEW_CONTRACT_ID> \
  --source <ADMIN_SECRET_KEY> \
  --network testnet \
  -- init \
  --token <TOKEN_ADDRESS> \
  --admin <ADMIN_ADDRESS>

# Verify version matches expected value
stellar contract invoke \
  --id <NEW_CONTRACT_ID> \
  --network testnet \
  -- version
# Expected output: <NEW_VERSION>

# Verify config
stellar contract invoke \
  --id <NEW_CONTRACT_ID> \
  --network testnet \
  -- get_config
```

### 2.3 Smoke-test critical paths on testnet

- [ ] `create_stream` — create a test stream and verify `get_stream_state`.
- [ ] `withdraw` — advance ledger time and withdraw; verify token balance.
- [ ] `cancel_stream` — cancel and verify refund.
- [ ] `version` — returns the new `CONTRACT_VERSION`.
- [ ] `get_config` — returns correct token and admin addresses.

### 2.4 Atomic WASM rotation (if replacing an existing testnet instance)

Soroban does not support in-place contract upgrades via `update_current_contract_wasm` in this protocol. The rotation is a **new deployment**:

```bash
# 1. Deploy new instance (see 2.1 above)
# 2. Initialize new instance (see 2.2 above)
# 3. Verify new instance (see 2.2 and 2.3 above)
# 4. Update all integrations to point at NEW_CONTRACT_ID
# 5. Announce OLD_CONTRACT_ID as deprecated
```

---

## Phase 3 — Mainnet cutover

### 3.1 Pre-mainnet gate

- [ ] All testnet smoke tests passed (Phase 2).
- [ ] WASM hash on testnet matches the locally built artifact:
  ```bash
  bash script/verify-wasm-checksum.sh --no-build
  ```
- [ ] At least one independent reviewer has audited the diff since the last deployed version.
- [ ] `docs/maintainer-security-checklist.md` has been worked through for this release.
- [ ] Notice window has elapsed (see Phase 1.5).

### 3.2 Deploy to mainnet

```bash
# Upload WASM to mainnet
stellar contract upload \
  --wasm target/wasm32-unknown-unknown/release/fluxora_stream.wasm \
  --source <DEPLOYER_SECRET_KEY> \
  --network mainnet

# Deploy contract instance
stellar contract deploy \
  --wasm-hash <WASM_HASH_FROM_UPLOAD> \
  --source <DEPLOYER_SECRET_KEY> \
  --network mainnet

# Initialize
stellar contract invoke \
  --id <NEW_CONTRACT_ID> \
  --source <ADMIN_SECRET_KEY> \
  --network mainnet \
  -- init \
  --token <TOKEN_ADDRESS> \
  --admin <ADMIN_ADDRESS>
```

### 3.3 Post-deployment verification

- [ ] Verify version:
  ```bash
  stellar contract invoke --id <NEW_CONTRACT_ID> --network mainnet -- version
  ```
- [ ] Verify config:
  ```bash
  stellar contract invoke --id <NEW_CONTRACT_ID> --network mainnet -- get_config
  ```
- [ ] Create a canary stream with a small deposit and verify `get_stream_state`.
- [ ] Confirm the WASM hash on-chain matches the CI artifact:
  ```bash
  bash script/verify-wasm-checksum.sh --no-build
  ```

---

## Phase 4 — In-flight stream handling during cutover

### 4.1 Inventory active streams on the old contract

```bash
# Get total stream count on old contract
stellar contract invoke --id <OLD_CONTRACT_ID> --network mainnet -- get_stream_count

# For each stream ID, check status
stellar contract invoke --id <OLD_CONTRACT_ID> --network mainnet \
  -- get_stream_state --stream_id <ID>
```

### 4.2 Recipient communication

- [ ] Notify all recipients with `Active` or `Paused` streams on the old contract.
- [ ] Provide a deadline by which they must withdraw accrued funds from the old contract.
- [ ] Provide instructions for the sender to recreate streams on the new contract if needed.

### 4.3 Cutover window

During the cutover window (between new contract deployment and old contract abandonment):

- Both contracts are live simultaneously.
- New streams are created only on the new contract.
- Old streams continue to accrue and can be withdrawn from the old contract.
- Senders may cancel old streams and recreate them on the new contract.

### 4.4 Old contract abandonment

- [ ] All active streams on the old contract have been resolved (withdrawn, cancelled, or completed).
- [ ] Announce the old `CONTRACT_ID` as permanently deprecated.
- [ ] Update all integrations (indexers, wallets, treasury tooling) to remove the old `CONTRACT_ID`.
- [ ] Archive the old contract address in `docs/upgrade.md`.

---

## Phase 5 — Post-upgrade monitoring

### 5.1 Monitoring thresholds (first 48 hours)

Monitor the following after mainnet deployment:

| Metric | Alert threshold |
|---|---|
| `create_stream` transaction failure rate | > 1% of attempts |
| `withdraw` transaction failure rate | > 1% of attempts |
| `version()` returns unexpected value | Any mismatch |
| Token balance of new contract | Unexpected decrease without corresponding withdrawals |
| Error events (`ContractError` variants) | Any `InvalidState` or `InvalidSignature` spike |

### 5.2 Rollback triggers

Initiate rollback (revert integrations to old contract) if any of the following occur within 48 hours of mainnet deployment:

- [ ] `version()` returns a value other than `CONTRACT_VERSION`.
- [ ] Any `create_stream` or `withdraw` call fails with an unexpected error on a valid input.
- [ ] Token balance discrepancy detected (contract holds more or fewer tokens than expected from stream accounting).
- [ ] A security vulnerability is reported that affects the new version.

### 5.3 Rollback procedure

Soroban contracts are immutable once deployed. "Rollback" means:

1. Revert all integrations to point at the **old `CONTRACT_ID`**.
2. Pause new stream creation on the new contract (via `set_contract_paused`).
3. Notify integrators of the rollback and the reason.
4. Investigate and fix the issue before attempting a new deployment.

---

## Checklist summary

| Phase | Gate |
|---|---|
| 1 — Pre-upgrade | `CONTRACT_VERSION` bumped, tests pass, WASM checksum updated, integrators notified |
| 2 — Testnet | New contract deployed, initialized, smoke-tested, WASM hash verified |
| 3 — Mainnet | Testnet gates passed, notice window elapsed, mainnet deployed and verified |
| 4 — Cutover | In-flight streams resolved, old contract deprecated |
| 5 — Monitoring | 48-hour monitoring window, rollback triggers defined |
