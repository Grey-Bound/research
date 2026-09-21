# BIP-322 address ownership binding

**GreyBound Research · 006**
**Status:** living note
**Applies to:** any wallet, exchange, app, or library that treats a BIP-322 (or “sign this message”) proof as evidence that a user controls a specific Bitcoin address

---

## 1. Intent

This note is the message-signing instance of the same rule as [001 §5.5](/Grey-Bound/research/blob/main/notes/001-wallet-app-threat-model.md), [004](/Grey-Bound/research/blob/main/notes/004-walletconnect-confirm-binding.md), and [005](/Grey-Bound/research/blob/main/notes/005-silent-payment-destination-binding.md): the cryptographic object accepted by the verifier must be bound to the claim the application is making.

For BIP-322, the claim is: *this party controls address A*.
A valid proof must show that the signature satisfies the scriptPubKey of A, not merely that some key in the witness can sign a challenge.

On 14 August 2026 the Rust crate `bip322` (`rust-bitcoin/bip322`) disclosed a critical verification flaw ([GHSA-5chw-87w3-j9cv](https://github.com/rust-bitcoin/bip322/security/advisories/GHSA-5chw-87w3-j9cv)). For P2WPKH and P2SH-P2WPKH, the verifier accepted a proof signed with an attacker-controlled key as proof of ownership of an unrelated victim address. P2TR was not affected.

| Address type | Affected | Notes |
| --- | --- | --- |
| P2WPKH | `>= 0.0.6, < 0.0.11` | Bug introduced with P2WPKH support (22 Aug 2024) |
| P2SH-P2WPKH | `>= 0.0.7, < 0.0.11` | Nested path added in 0.0.7 |
| P2TR | not affected | — |

First patched release: **0.0.11** (2 Aug 2026; fix commit `e8accbe`).
Current release as of 21 Sep 2026: **0.0.12** (31 Aug 2026; adds key-mismatch regression tests and hardening, `#81`).
Versions 0.0.6–0.0.10 were yanked from crates.io. No CVE assigned. Do not confuse this crate with the unrelated fork `bip322-rs`.

This note extracts the engineering property. It is not a full vendor autopsy and does not claim every BIP-322 implementation is broken. It is a review checklist for anyone who uses address-ownership proofs for withdrawals, account linking, borrowing, or similar state changes.

Primary source: [rust-bitcoin/bip322 security advisory](https://github.com/rust-bitcoin/bip322/security/advisories/GHSA-5chw-87w3-j9cv) (14 Aug 2026). Spec: [BIP 322](https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki).

---

## 2. What BIP-322 is supposed to prove

BIP-322 defines a generic signed-message format built from two virtual transactions:

| Object | Role |
| --- | --- |
| to_spend | Commitment to the message (`message_hash`) and the address being proven (`message_challenge` = that address’s scriptPubKey) |
| to_sign | Transaction that spends `to_spend`; its witness / scriptSig is the `message_signature` |

Verification succeeds only if `message_signature` satisfies `message_challenge`. That is the binding: the signature is not free-floating; it must unlock the script of the claimed address.

Current spec variants (signers MUST prefix the encoding):

| Variant | Prefix | Payload |
| --- | --- | --- |
| Simple | `smp` | witness stack |
| Full | `ful` | full `to_sign` transaction |
| Full (proof of funds) | `pof` | finalized PSBT of `to_sign` |
| Legacy (BIP-137) | none | compact recoverable ECDSA; restricted to P2PKH |

A verifier MAY assume *simple* only for pre-finalization compatibility when a prefix is absent. Do not mix verifiers across variants.

Proof-of-funds extends the idea to additional real UTXOs. The ownership rule is the same: the challenge path must be bound to the address (or scripts) the application believes it is authenticating.

Applications that skip this binding and only check “signature verifies under some pubkey in the proof” are not doing BIP-322 verification. They are doing a weaker, unsafe check.

---

## 3. Pattern: signature valid, ownership false

### What happened in bip322 (P2WPKH path)

For affected versions, verification roughly:

1. Took the public key from the caller-controlled witness
2. Checked that the signature was valid for that key
3. Compared “expected key” to the key from the same witness (tautology)
4. Never checked that `HASH160(pubkey)` (or the nested P2SH form) produced the claimed address’s scriptPubKey

Concrete form of the tautology in `verify_full_p2wpkh`: `pub_key` was parsed from the witness, then compared back to that same witness element as `PublicKeyMismatch`. The comparison could not fail for a well-formed attacker witness.

So an attacker could:

1. Pick any victim P2WPKH / P2SH-P2WPKH address
2. Build the BIP-322 challenge for that address and an arbitrary message
3. Sign with their own key
4. Submit the proof and pass verification

No victim key material. No victim interaction. Application-level nonces do not help: the attacker signs whatever fresh challenge the server issues.

All four public APIs hit the same path: `verify_simple`, `verify_simple_encoded`, `verify_full`, `verify_full_encoded`.

### Why the class is broader than one crate

Any verifier that:

- treats witness-embedded keys as authoritative, or
- verifies a sighash without reconstructing / matching the challenge scriptPubKey from the claimed address, or
- confuses “valid ECDSA/Schnorr under key K” with “K controls address A”, or
- accepts a BIP-322-shaped object without the script / sighash constraints the spec requires (example of the last: [GHSA-xq4h-wqm2-668w](https://github.com/advisories/GHSA-xq4h-wqm2-668w) in Babylon — `SIGHASH_ALL` not enforced)

can reproduce the same failure mode. Libraries, custom Go/JS ports, app-level wrappers, and forks (`bip322-rs` and others) are all in scope for review.

---

## 4. What the disclosure fixed (and what apps must still do)

Per the advisory, 0.0.11 binds the witness public key to the challenged address before accepting the signature: derive the expected scriptPubKey from the witness key (P2WPKH / nested P2SH) and require equality with the scriptPubKey of the claimed address; otherwise `PublicKeyMismatch`. The witness key must be compressed.

That closes the crate bug. 0.0.12 then adds explicit key-mismatch tests and extra hardening (`#81`) — the regression the review checklist already required.

Application owners still must:

1. Upgrade to a patched verifier (**pin `>= 0.0.12`** for this crate; `0.0.11` is the minimum patched line) or reimplement verification correctly
2. Not treat “we call `verify_*`” as safety if the dependency is pinned to a yanked range, to an unpatched fork, or to a pre-0.0.11 lockfile that `cargo audit` may still miss if no RustSec ID was published
3. Audit every code path that maps a successful BIP-322 (or legacy signed-message) check to a privileged action

---

## 5. Required property

Commitment: a successful ownership verification implies that the proof’s `message_signature` satisfies the scriptPubKey of the exact address the application is authenticating for that message.

| Check | Expectation |
| --- | --- |
| Challenge construction | `message_challenge` is the scriptPubKey of the claimed address (from the address string the app already trusts for this session) |
| Key ↔️ address | Witness / revealed keys must hash or compile to that same scriptPubKey (type-appropriate: P2WPKH, nested, P2TR, multisig policy, etc.) |
| Signature | Valid under the script rules for that challenge, with the message hash specified by BIP-322 (including sighash and variant-prefix rules) |
| Failure | Any mismatch → reject; no partial “signature ok, address soft-fail” |
| App binding | Privileged actions (withdraw, link, credit) only after reject-by-default verification for the same address and message the app displayed or stored |

Legacy BIP-137-style “sign message” flows have their own format risks; do not mix verifiers. If the product says BIP-322, verify BIP-322.

---

## 6. What to check in review

Trace one “prove you own this address” flow end-to-end: UI address → challenge message → client proof → server/library verify → privileged action.

1. Which library and exact version verify the proof? Is it `bip322` or a similarly named fork?
2. For P2WPKH / P2SH-P2WPKH: does verification derive scriptPubKey from the claimed address and require the witness key to match it?
3. Can you forge a proof with an unrelated key against a fixed victim address in a local test?
4. Are P2TR, multisig, P2WSH, and (if implemented) P2PKH / legacy paths covered by the same binding rule, or only the happy path?
5. Is the message the user saw the same message hashed into the proof (no silent prefix / domain separation bugs)? Does the encoded proof use the variant prefix the app thinks it is verifying (`smp` / `ful` / `pof` / legacy)?
6. After a failed verify, is the action blocked, or is there a fallback to a weaker check?
7. Dependency pins: any `bip322 >= 0.0.6, < 0.0.11` (or equivalent unpatched ports / forks)? Prefer `>= 0.0.12`.
8. Are there multiple verifiers (mobile, backend, partner API) with inconsistent strictness?
9. Does CI contain the `#81`-style case: attacker key + victim address → reject, for each supported address type?

Hostile assumption: the prover chooses the witness and the signature. The only authority for “which address?” is the address the application already bound to the session, plus correct script satisfaction.

---

## 7. Hardening direction

- Construct `to_spend` from your claimed address + message; do not let the prover supply a different challenge scriptPubKey.
- Verify script satisfaction against that challenge; then (where the script type requires) bind revealed keys to the address policy.
- Fail closed on type mismatch, nested-script mismatch, unknown script classes, missing/wrong variant prefix, or non-standard sighash.
- Pin patched libraries (`bip322 >= 0.0.12` today); add a regression test: *unrelated key + victim address → reject*.
- Separate message identity (what was signed) from address identity (who must be able to sign); both must hold.
- For high-value actions, prefer challenge messages that are tightly scoped (action, amount, destination, expiry) so a stolen proof is less useful even if verification were weak historically.

---

## 8. Control catalog

| ID | Control |
| --- | --- |
| O1 | Challenge scriptPubKey taken only from the app’s claimed address |
| O2 | Signature must satisfy that challenge (BIP-322 rules, including variant prefix and sighash) |
| O3 | Witness / revealed keys bound to that address (no tautological self-compare) |
| O4 | Unrelated-key forgery test in CI for each supported address type |
| O5 | Patched verifier dependency; yanked ranges and unpatched forks blocked (`bip322 >= 0.0.12`) |
| O6 | Privileged actions require successful verify of the same address + message |
| O7 | No fallback to weaker legacy verify on BIP-322 failure |

---

## 9. Self-assessment

- [ ] Ownership flow traced from claim → proof → action
- [ ] P2WPKH and nested P2WPKH cannot be forged with an external key
- [ ] P2TR / multisig / P2WSH / legacy paths documented or explicitly out of scope
- [ ] Library is `rust-bitcoin/bip322` `>= 0.0.12`, or an independently reviewed port — not a yanked range and not an unpatched fork
- [ ] Regression test: attacker key, victim address → reject
- [ ] Message bytes authenticated equal message bytes shown or stored
- [ ] Encoded variant (simple / full / pof / legacy) matches what the product claims to verify

---

## 10. Related notes

- [001 — Wallet application threat model](/Grey-Bound/research/blob/main/notes/001-wallet-app-threat-model.md) (§5.5 preview → sign)
- [004 — WalletConnect confirm binding](/Grey-Bound/research/blob/main/notes/004-walletconnect-confirm-binding.md) (signed object vs UI claim)
- [005 — Silent Payment destination binding](/Grey-Bound/research/blob/main/notes/005-silent-payment-destination-binding.md) (destination claim vs constructed output)

Adjacent BIP-322 compliance bug (different root cause, same “accepted object ≠ spec claim” class): [GHSA-xq4h-wqm2-668w](https://github.com/advisories/GHSA-xq4h-wqm2-668w) (Babylon, missing `SIGHASH_ALL` enforcement).

---

## 11. Residual risk

After correct binding, a BIP-322 proof shows the prover could satisfy the address script for that message at proof time. It does not by itself prove ongoing control of on-chain funds, absence of shared keys, or safety of the application’s business logic. Proof-of-funds and UTXO checks are separate.

Yank + GHSA does not guarantee that every consumer lockfile was upgraded, nor that `cargo audit` flags the range if a RustSec advisory was never filed. Treat old pins as still live until proven otherwise.

This note is not a warranty that any particular library or product is free of bugs.

---

## References

- [GHSA-5chw-87w3-j9cv — bip322 address-ownership bypass](https://github.com/rust-bitcoin/bip322/security/advisories/GHSA-5chw-87w3-j9cv)
- [bip322 0.0.12](https://github.com/rust-bitcoin/bip322/releases/tag/0.0.12) — key mismatch tests and hardening (`#81`)
- [BIP 322 — Generic Signed Message Format](https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki)
- GreyBound Research [001](/Grey-Bound/research/blob/main/notes/001-wallet-app-threat-model.md) / [004](/Grey-Bound/research/blob/main/notes/004-walletconnect-confirm-binding.md) / [005](/Grey-Bound/research/blob/main/notes/005-silent-payment-destination-binding.md) — commitment boundaries in wallets and verifiers
