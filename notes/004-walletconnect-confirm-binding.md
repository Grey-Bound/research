# WalletConnect confirm binding

**GreyBound Research · 004**  
**Status:** living note  
**Applies to:** wallets, extensions, and desktop signers that approve EVM (and similar) transactions over WalletConnect or any comparable dapp session / in-app browser RPC

---

## 1. Intent

This note is the EVM-session instance of [001 §5.5](001-wallet-app-threat-model.md): preview → sign is the critical boundary.

The user is shown a confirm sheet. The wallet signs an object that sheet does not fully describe. The missing piece is usually calldata. Sometimes it is native `value`. Sometimes it is an allowance the wallet rewrites after the sheet is drawn.

[101](101-psbt-mistakes.md) states the Bitcoin form: preview is not a commitment. Same rule here. Session request methods that return a signed payload the dapp can broadcast are in scope (`eth_sendTransaction`, `eth_signTransaction`, and equivalents). Typed-data (`eth_signTypedData`) is the same class of failure; it is not expanded in this note.

This is not a WalletConnect protocol bug. Pairing and session approve can be correct. The hole is the wallet’s confirm → sign glue.

## 2. Pattern

A paired dapp sends `eth_sendTransaction` (or `eth_signTransaction`) with:

- `to` — often a token contract
- `value` — often `0`
- `data` — `transfer` / `approve` / `setApprovalForAll` / arbitrary call

The wallet then does some mix of the following.

### 2.1 Confirm state drops `data`

1. Persist `amount` from native `value` and `address` from `to`.
2. Render that pair (sometimes gas).
3. On confirm, spread the original RPC `params[0]` into the signer — including `data`.

The sheet is a zero native send to a contract. The signed tx moves tokens or sets allowance.

WC is often enabled only on the native coin. That native wallet is still what signs `data`. A session on `eip155:*` can `transfer` / `approve` every token on that chain while the UI shows `0` of the native asset.

### 2.2 ABI parse, then discard the money fields

The wallet already decodes ERC-20 `transfer` / `approve`. Named in-app methods (`execute`, `pay`, swap helpers) keep an amount. The default branch does not set one. Confirm amount becomes `Number(x) || 0`.

`data` is still copied onto the signed object.

The sheet may show an ellipsized hex line. That is not a confirmation of amount, recipient, or spender. `to` on the sheet is the token contract, not the decoded `transfer` recipient or `approve` spender.

In-app swap flows that whitelist spenders do not cover this path. Session requests are a separate builder.

### 2.3 Display object ≠ signed object

`eth_sendTransaction` may build a local tx proposal (and a backend may force native `value` to 0). Confirm binds that proposal.

`eth_signTransaction` often signs `params[0]` with a local signer and returns the raw tx to the dapp. Native `value` in the request is in the signed blob even when the send path would have zeroed it. The dapp broadcasts.

If proposal build throws, some wallets still enable swipe and fall through to the raw-params signer. Fail-open.

### 2.4 Sheet amount is not the encoded amount

Selector `approve` is decoded for the popup. The sheet prints the dapp’s allowance. On confirm the wallet re-encodes `approve(spender, max)` and signs that.

The user confirmed a finite allowance. The chain got unlimited.

### 2.5 Simulation as the only amount source

Primary amount (and sometimes the whole risk banner) is taken from a simulator / third-party check, not from local decode of `to` / `value` / `data`.

If the mapper throws on valid tx JSON (nested `accessList`, numeric fields), or the check returns error, the sheet is empty or fee-only. Sign still proceeds.

A screenless hardware signer that only receives hashes cannot correct that sheet.

## 3. Why it matters

Self-custody fails when the interface tells one story and the transaction tells another.

For ERC-20, economic impact lives in calldata. A zero native send with `transfer(attacker, N)` or `approve(spender, max)` in `data` is a token drain or an unlimited operator, not a no-op.

Severity is High, not Critical: the user still confirms and unlocks the key (PIN, password, biometric, card tap). It is not an unauthenticated remote drain. It is a blind-sign class issue. Session must already be approved.

## 4. Required property

**Commitment:** every field that affects the signed payload is on the confirm sheet in a form the user can understand, or the wallet refuses to sign.

The object hashed / broadcast is the object the sheet was rendered from. If that object changes, approval is void.

| Field | On the sheet |
| --- | --- |
| Destination | Yes. Token contract vs recipient/spender, labelled. |
| Native `value` | Yes. The signed `value`, not a separate proposal that gets dropped. |
| Calldata | Yes. Hex minimum. Decoded method + args when the ABI is known. |
| ERC-20 `transfer` / `transferFrom` | token, amount, recipient |
| ERC-20 `approve` / `setApprovalForAll` | token, spender/operator, amount or **unlimited** |
| Unknown calldata | refuse. Do not confirm `0` and attach hidden `data`. |
| Simulation / malware check | advisory. Never the only amount source. Error → no sign. |

Do not attach `data` to the signed object unless every semantic field it encodes is visible and bound to that approval.

Do not rewrite calldata after the sheet (finite approve → max) unless the sheet already says unlimited and the user confirmed that.

## 5. What to check in review

Trace `eth_sendTransaction` and `eth_signTransaction` separately: session request → confirm state → UI → sign → broadcast / WC respond.

1. Is `params[0].data` copied into the signed tx?
2. Does confirm state store `data`, or only native amount + `to`?
3. If ERC-20 ABI is parsed: are amount and spender/recipient kept through confirm, or discarded?
4. Is `to` on the sheet the token contract while the recipient lives in `data`?
5. Is truncated hex treated as “calldata shown”?
6. Can the user approve when decode or simulation fails but `data` is still attached?
7. Does `eth_signTransaction` sign `params[0]` (including native `value`) while the sheet uses a different proposal?
8. On approve: is the encoded allowance the number on the sheet?
9. Is trust placed in dapp name/url/icons, or in a simulator, instead of the signed fields?
10. After broadcast, does the session response correspond to the full signed payload?

**Fixtures (CI, sheet text + signed payload):**

- `value = 0`, `data = transfer(recipient, N)` → sheet is not a harmless zero send; signed `data` matches decoded recipient and `N`.
- `value = 0`, `data = approve(spender, max)` → sheet says unlimited; spender is explicit.
- `value = 0`, unknown calldata → sign refused.
- Finite approve on the sheet → signed allowance is that finite value, not max.
- `eth_signTransaction` with non-zero `value` → sheet shows that value; signed `value` matches.

Same checks on any in-app browser that shares the confirm component.

Other namespaces: if confirm binds the first obvious transfer instruction while extra instructions stay in the signed bytes, that is the same bug.

## 6. Hardening direction

- After ABI parse, keep method-specific fields for confirm. Do not decode and then throw them away.
- Primary line: method + token + amount (or **Unlimited**).
- Recipient line: decoded recipient/spender — not only the token contract.
- Bind the signer to the confirmed struct (or a hash of it). Re-read `params[0]` at click is how 2.1 and 2.3 happen.
- Unknown or failed decode: refuse. Do not show `0` and still attach `data`.
- Simulator errors are unsafe, not silent.
- `eth_sendTransaction` and `eth_signTransaction` use the same binding rule. A backend that zeroes native `value` on send does not protect tokens and does not protect `signTransaction`.
- Screenless hardware: the host sheet is the only WYSIWYS. Empty sheet → no hash to the card.

## 7. Control catalog

| ID | Control |
| --- | --- |
| W1 | Confirm fields derived from the object that will be signed |
| W2 | Calldata visible; decoded when the wallet already has the ABI |
| W3 | Token amount and spender/recipient explicit for ERC-20 methods |
| W4 | No sign when those fields cannot be shown |
| W5 | Dapp metadata and simulators never substitute for field binding |
| W6 | Approve encoding matches the sheet (no silent max rewrite) |
| W7 | `eth_signTransaction` bound the same way as `eth_sendTransaction` |
| W8 | Tests: zero native + transfer/approve; unknown data; finite vs max approve |

## 8. Self-assessment

- `eth_sendTransaction` and `eth_signTransaction` documented end-to-end
- Confirm UI includes calldata or a structured decode (not hex-as-decoration)
- Approve/transfer fixtures in CI (sheet text + signed payload)
- Unknown calldata cannot be approved as a zero send
- Simulation/mapper failure cannot enable sign
- In-app browser sharing that confirm component reviewed on the same fixtures
- Other session methods that return a signed tx checked for the same gap

## 9. Related notes

- [001 — Wallet application threat model](001-wallet-app-threat-model.md) (§5.5 preview → sign; §5.6 sessions)
- [101 — PSBT mistakes](101-psbt-mistakes.md) (preview is not a commitment)

## References

- WalletConnect session request methods (`eth_sendTransaction`, `eth_signTransaction`, and related)
- ERC-20 `transfer` / `approve` / `setApprovalForAll` (amount and spender/recipient in calldata)
- GreyBound Research 001 — preview → sign as the highest-value wallet boundary
