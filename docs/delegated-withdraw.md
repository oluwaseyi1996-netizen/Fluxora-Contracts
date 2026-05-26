# Delegated Withdraw — Signature Design Rationale and Replay-Protection Scheme

Reference for integrators, auditors, and relayer implementors.

**Source of truth:** `contracts/stream/src/lib.rs` — `delegated_withdraw` entry-point (search `pub fn delegated_withdraw`).  
**Cross-references:** [docs/audit.md](./audit.md) · [docs/streaming.md](./streaming.md) · [docs/security.md](./security.md)

---

## 1. Why delegated withdraw exists

The standard `withdraw` entry-point requires the recipient to submit the Stellar transaction themselves, paying the network fee. This is a friction point for:

- **Gas-less UX**: recipients who hold no XLM for fees.
- **Automated treasury tooling**: relayers that batch withdrawals on behalf of many recipients.
- **Mobile / custodial wallets**: where the signing key is available but the account may not be funded.

`delegated_withdraw` lets a **relayer** submit the transaction and pay the fee, while the **recipient** authorises the withdrawal off-chain with an Ed25519 signature. The relayer receives no tokens; all funds go to the `destination` address specified in the signed message.

---

## 2. Why Ed25519 over Soroban-native auth

Soroban's native `require_auth` / `require_auth_for_args` mechanism is the preferred authorisation path for most entry-points in this contract. For delegated withdrawal, it was not used because:

| Concern | Soroban-native auth | Ed25519 off-chain signature |
|---|---|---|
| Recipient must be online to sign the Stellar transaction | Yes | No — signs a raw message offline |
| Signature can be pre-computed and handed to a relayer | No | Yes |
| Replay protection built into the protocol | Via sequence number (per-account) | Via per-recipient nonce stored in contract |
| Works with non-Stellar signing environments (JS, hardware wallets) | Requires Stellar XDR tooling | Standard Ed25519 — widely supported |
| Auditable message format | Opaque to off-chain tooling | Explicit byte layout documented here |

The trade-off is that the contract must manage its own nonce storage and message construction, which this document specifies precisely.

---

## 3. Signed message byte layout

The recipient signs the **SHA-256 hash** of the following concatenated byte string:

```
"fluxora_delegated_withdraw"   (26 bytes, UTF-8, no null terminator)
|| contract_address_xdr        (variable, XDR-encoded Soroban Address)
|| destination_xdr             (variable, XDR-encoded Soroban Address)
|| stream_id                   (8 bytes, u64 big-endian)
|| nonce                       (8 bytes, u64 big-endian)
|| deadline                    (8 bytes, u64 big-endian)
```

The 32-byte SHA-256 digest is then verified against the 64-byte Ed25519 `signature` using the recipient's public key extracted from their Stellar G-address.

### Field semantics

| Field | Type | Purpose |
|---|---|---|
| `"fluxora_delegated_withdraw"` | ASCII prefix | Domain separator — prevents cross-protocol signature reuse |
| `contract_address_xdr` | XDR bytes | Binds the signature to this specific contract instance; prevents replay on a different deployment |
| `destination_xdr` | XDR bytes | Specifies where tokens are sent; relayer cannot redirect funds |
| `stream_id` | u64 big-endian | Binds the signature to a specific stream |
| `nonce` | u64 big-endian | Monotonic counter; prevents replay of the same signature |
| `deadline` | u64 big-endian | Ledger timestamp expiry; limits the window a signature is valid |

---

## 4. Nonce lifecycle

Nonces are stored in **persistent contract storage** under `DataKey::WithdrawNonce(recipient: Address)`.

| Event | Nonce value |
|---|---|
| Recipient has never used `delegated_withdraw` | `0` (default; key absent from storage) |
| Successful `delegated_withdraw` call (amount > 0) | Incremented by 1 |
| `delegated_withdraw` call with 0 withdrawable amount | **Not incremented** — nonce is preserved |
| Failed call (wrong nonce, expired deadline, bad signature) | Not incremented |

Key properties:
- **Strictly monotonic**: nonces never decrease or skip.
- **Per-recipient**: each recipient has an independent counter; one recipient's nonce does not affect another's.
- **No reset**: there is no admin function to reset a nonce. Once consumed, a nonce value is permanently invalid.
- **Read via `get_withdraw_nonce(recipient)`**: integrators should call this view entry-point to fetch the current nonce before constructing a signature.

---

## 5. Deadline semantics

`deadline` is a **ledger timestamp** (Unix seconds, same unit as `env.ledger().timestamp()`).

- The signature is valid if `env.ledger().timestamp() <= deadline` at the moment the transaction is executed.
- If `env.ledger().timestamp() > deadline`, the call returns `ContractError::SignatureDeadlineExpired`.
- There is no minimum deadline; a deadline in the past is immediately rejected.
- Recommended practice: set `deadline = current_ledger_timestamp + 300` (5 minutes) for interactive flows, or longer for batch relayer pipelines.

---

## 6. Public key derivation

The contract derives the recipient's Ed25519 public key from their Stellar G-address XDR encoding:

```
Stellar G-address XDR layout:
  4 bytes  — type discriminant (PublicKeyType)
  4 bytes  — key type discriminant (ED25519)
  32 bytes — raw Ed25519 public key bytes
```

The contract reads the last 32 bytes of the XDR-encoded recipient address as the public key. This is valid for all standard Stellar accounts (G-addresses). Contract addresses (C-addresses) are not valid recipients for `delegated_withdraw` because their XDR layout differs.

---

## 7. Client-side message construction

### JavaScript (Stellar SDK)

```javascript
import { xdr, hash, Keypair, Address } from "@stellar/stellar-sdk";
import { Buffer } from "buffer";

/**
 * Build the message bytes and sign them for delegated_withdraw.
 *
 * @param {string} contractId   - Bech32 contract address (C-address)
 * @param {string} destination  - Bech32 destination address
 * @param {bigint} streamId     - Stream ID (u64)
 * @param {bigint} nonce        - Current recipient nonce from get_withdraw_nonce()
 * @param {bigint} deadline     - Ledger timestamp expiry (Unix seconds)
 * @param {Keypair} recipientKeypair - Recipient's Ed25519 keypair
 * @returns {Buffer} 64-byte Ed25519 signature
 */
function signDelegatedWithdraw(
  contractId,
  destination,
  streamId,
  nonce,
  deadline,
  recipientKeypair
) {
  const prefix = Buffer.from("fluxora_delegated_withdraw", "utf8");

  // XDR-encode addresses (Soroban Address XDR)
  const contractXdr = Buffer.from(
    new Address(contractId).toScVal().toXDR()
  );
  const destXdr = Buffer.from(
    new Address(destination).toScVal().toXDR()
  );

  // u64 big-endian helpers
  const u64BE = (n) => {
    const buf = Buffer.alloc(8);
    buf.writeBigUInt64BE(n);
    return buf;
  };

  const msg = Buffer.concat([
    prefix,
    contractXdr,
    destXdr,
    u64BE(streamId),
    u64BE(nonce),
    u64BE(deadline),
  ]);

  const msgHash = hash(msg); // SHA-256
  return recipientKeypair.sign(msgHash); // 64-byte Ed25519 signature
}
```

### Rust (test / relayer)

```rust
use ed25519_dalek::{Signer, SigningKey};
use sha2::{Digest, Sha256};
use stellar_xdr::curr::{ScAddress, WriteXdr};

fn sign_delegated_withdraw(
    contract_xdr: &[u8],   // XDR bytes of contract address
    destination_xdr: &[u8], // XDR bytes of destination address
    stream_id: u64,
    nonce: u64,
    deadline: u64,
    signing_key: &SigningKey,
) -> [u8; 64] {
    let mut msg = Vec::new();
    msg.extend_from_slice(b"fluxora_delegated_withdraw");
    msg.extend_from_slice(contract_xdr);
    msg.extend_from_slice(destination_xdr);
    msg.extend_from_slice(&stream_id.to_be_bytes());
    msg.extend_from_slice(&nonce.to_be_bytes());
    msg.extend_from_slice(&deadline.to_be_bytes());

    let hash = Sha256::digest(&msg);
    signing_key.sign(&hash).to_bytes()
}
```

> **Note:** The Soroban contract uses `env.crypto().sha256()` which produces a standard SHA-256 digest. The client must hash the raw message bytes (not double-hash).

---

## 8. Security assumptions

| Assumption | Consequence if violated |
|---|---|
| The recipient's Ed25519 private key is kept secret | An attacker with the key can drain the stream to any destination |
| The `deadline` is set to a short window | A leaked signature can be replayed until the deadline expires |
| The `nonce` is fetched fresh before signing | Signing with a stale nonce produces an immediately-invalid signature |
| The `contract_address` in the message matches the deployed contract | A signature cannot be replayed on a different contract instance |
| The `destination` in the message is the intended recipient | The relayer cannot redirect funds — destination is bound in the signature |
| The relayer submits the transaction before `deadline` | Delayed submission causes `SignatureDeadlineExpired` |

### What the contract does NOT protect against

- **Relayer censorship**: a relayer can refuse to submit a valid signature. Recipients should use multiple relayers or fall back to `withdraw` directly.
- **Front-running**: a relayer can observe a signed message in the mempool and submit it first. Since the destination is fixed in the signature, front-running only affects timing, not fund destination.
- **Key compromise**: if the recipient's private key is compromised, all future delegated withdrawals are at risk. There is no revocation mechanism; the recipient should withdraw directly via `withdraw` to drain the stream.

---

## 9. ed25519-dalek version

The contract uses Soroban's built-in `env.crypto().ed25519_verify` (part of `soroban-sdk 21.7.7`), which delegates to the Stellar host's Ed25519 implementation. No direct `ed25519-dalek` dependency is used in the contract itself.

For **test and relayer code** (dev-dependencies), `ed25519-dalek 2.1` is pinned in `contracts/stream/Cargo.toml`:

```toml
ed25519-dalek = { version = "2.1", features = ["rand_core"] }
```

**Breaking API changes in ed25519-dalek 2.x vs 1.x:**
- `Keypair` was renamed to `SigningKey` / `VerifyingKey`.
- `sign` now takes `&[u8]` directly (no pre-hashing required by the library; the contract pre-hashes with SHA-256 before calling `ed25519_verify`).
- `from_bytes` now returns `Result` instead of panicking.

If upgrading `ed25519-dalek`, update test helpers accordingly and re-run `cargo test -p fluxora_stream`.

---

## 10. Entry-point reference

```
delegated_withdraw(
    env:        Env,
    stream_id:  u64,
    relayer:    Address,   // pays tx fee; requires_auth
    destination: Address,  // receives tokens; must not be contract address
    nonce:      u64,       // must equal get_withdraw_nonce(recipient)
    deadline:   u64,       // ledger timestamp; must be >= current timestamp
    signature:  BytesN<64> // Ed25519 signature over SHA-256(message)
) -> Result<i128, ContractError>
```

**Returns:** amount transferred (0 if nothing to withdraw; nonce not consumed).

**Errors:**

| Error | Condition |
|---|---|
| `SignatureDeadlineExpired` | `current_timestamp > deadline` |
| `InvalidParams` | nonce mismatch, or `destination == contract_address` |
| `InvalidSignature` | Ed25519 verification fails (host trap / transaction revert) |
| `StreamNotFound` | stream does not exist |
| `InvalidState` | stream is `Completed` or `Paused` |
