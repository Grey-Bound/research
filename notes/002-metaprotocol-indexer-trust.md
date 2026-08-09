# Metaprotocol and indexer trust

**GreyBound Research · 002**  
**Status:** living note  
**Applies to:** Ordinals / BRC-20 / Runes-class wallets, marketplaces, indexers, and APIs that influence what users sign

---

## 1. Intent

Metaprotocol products fail in a distinctive way: Bitcoin enforces one thing, the product UI asserts another, and the user signs believing they are the same.

This note is the trust model Greybound uses when reviewing those systems. It assumes familiarity with PSBTs and UTXOs ([001](001-wallet-app-threat-model.md)). It does not argue whether a metaprotocol “should” exist. It asks what happens when the opinion layer is wrong, lagged, forked, or hostile.

## 2. The stack you actually ship

```text
┌──────────────────────────────────────────────────────────┐
│ Product UI / marketplace copy / push notifications       │
├──────────────────────────────────────────────────────────┤
│ Indexer API + caches + “repair” admin                    │
├──────────────────────────────────────────────────────────┤
│ Indexing rules (inscription parsing, ticker ledgers…)    │  ← convention
├──────────────────────────────────────────────────────────┤
│ Mempool + chain                                          │
├──────────────────────────────────────────────────────────┤
│ Script, amounts, outpoints, signatures                   │  ← Bitcoin
└──────────────────────────────────────────────────────────┘
```

Only the bottom layer is consensus-enforced. Indexing rules are social/technical conventions implemented in software. Caches make them stale. Admin tools make them mutable. UIs make them look like truth.

A review that only checks “do we parse inscriptions like ord?” and skips sign-time binding will miss the losses users actually take.

## 3. Assets specific to this domain

| Asset | Why it matters |
|--------|----------------|
| Inscribed / rare sat outpoints | Easy to burn as fees if unlabeled |
| Ticker balances (indexer ledger) | User belief ≠ re-derivable from chain without ruleset |
| Marketplace orders / escrow PSBTs | Multi-party templates; abort and partial-fill risk |
| Reveal / commit linkages | Funded commit with wrong or missing reveal |
| Indexer admin capability | Silent rewrite of the opinion layer |
| API keys for write/reindex | Often equivalent to admin |

## 4. Trust boundaries

### 4.1 Indexer → display

**Required property:** anything shown as “you own X” or “balance is Y” is labeled as indexer-sourced when it is not locally proven, and is never the sole input to a sighash field.

**Failures:**

- Single API, no age/reorg indicator
- Two indexers disagree; client hard-codes one
- Cached balances after reorg or failed reindex
- “Pending” states shown as final

**Evidence:** API contract includes cursor/block height; UI shows sync tip; client tests for stale snapshot rejection on sign.

### 4.2 Indexer → PSBT build

**Required property:** the client can rebuild the economic intent locally. Server templates are suggestions until re-encoded and re-validated.

**Failures:**

- Server supplies full PSBT; client shows a summary string and signs
- Decimals / divisibility taken from API without binding to ticker deployment rules the client implements
- Fee and postage outputs inserted by server without line-item preview
- Input selection that spends inscribed outpoints without local inscription database check

**Evidence:** given chain data + local ruleset, client produces the same PSBT (or rejects). Golden vectors for transfer/reveal/deploy shapes you claim to support.

### 4.3 Marketplace settlement

**Required property:** every party’s best/worst abort path is defined. Funds cannot be claimed by the operator through a “cancel” that is actually a spend.

**Failures:**

- Unsigned residual paths in partially signed templates
- Timeouts that favor one side without disclosure
- Operator key that can finalize either direction
- Off-chain match with on-chain settle where replace-by-fee races are ignored

**Evidence:** state machine diagram; PSBT role matrix; tests for cancel, expire, partial fill, double-claim.

### 4.4 Wallet coin selection under metaprotocol constraints

**Required property:** outputs with metaprotocol significance are tagged locally (or refused) before fee burning / consolidation.

**Failures:**

- Send-max burns inscription postage
- Auto-consolidate merges rare sats
- “Simple BTC send” mode ignores local inscription index

### 4.5 Admin and write APIs

**Required property:** reindex, repair, and balance overrides are authenticated, audited, and environment-separated.

**Failures:**

- Shared prod/staging admin tokens
- Unauthenticated repair endpoints on internal network (reachable via SSRF)
- Silent balance edits without append-only audit log

## 5. Scenario classes

### S1 — False ownership

Indexer attributes an inscription to the wrong address after a reorg or bug. Marketplace allows list/sell. Buyer pays; seller never could deliver. Or worse: wallet offers a “transfer” PSBT that burns the asset.

### S2 — Decimal / ticker confusion

UI shows “1000 TOKEN”. Ruleset divisibility means the PSBT moves a different on-chain representation. User approves based on UI string.

### S3 — Server-authored transfer

Wallet calls `POST /build-transfer`, receives PSBT, shows “Transfer 1 inscription to bc1q…”. PSBT pays the marketplace fee address twice or substitutes destination. No local rebuild.

### S4 — Commit without reveal

User funds a commit. Reveal path broken or key held by operator. Funds stuck or operator-gated.

### S5 — Inscription-as-fee

UTXO view from BTC-only node path. Coin selection pays fees with inscribed sat. Irreversible.

### S6 — Admin rewrite

Compromised indexer admin sets attacker balance, issues withdrawals against a custody or hybrid model, or poisons caches all light clients read.

## 6. Control catalog

| ID | Control |
|----|---------|
| M1 | Sign-time local revalidation of metaprotocol claims that affect outputs |
| M2 | UI provenance: source, block height, reorg depth policy |
| M3 | Local inscription / asset tags for coin selection |
| M4 | No signing of server PSBTs without canonical rebuild or byte-level preview |
| M5 | Marketplace PSBT role matrix + abort tests |
| M6 | Dual-indexer divergence alarms for operators |
| M7 | Admin audit log (append-only) for repair/override |
| M8 | SSRF-safe egress from indexer workers (see [003](003-bitcoin-app-backend-hardening.md)) |
| M9 | Explicit postage/fee line items in preview |
| M10 | Version-pin indexing rules; document break-glass reindex |

## 7. Review method

1. Inventory every metaprotocol action (deploy, mint, transfer, list, buy, reveal, send-BTC).
2. For each action, mark which fields come from chain, indexer, or server template.
3. Trace those fields into the PSBT. Anything that hits sighash without local check is a finding candidate.
4. Break the indexer (stale, empty, contradictory, hostile fixtures) and watch the client.
5. Exercise marketplace abort paths with real PSBT fixtures.
6. Review admin authn/z and audit logs like a banking ledger, not a debug tool.

## 8. Self-assessment

- [ ] Written ruleset version + compatibility policy  
- [ ] Client can explain ownership from chain+rules without the marketplace  
- [ ] Hostile indexer fixtures in CI for sign paths  
- [ ] Inscription-aware coin selection (or explicit unsafe mode)  
- [ ] Marketplace state machine documented and tested  
- [ ] Admin actions audited and alerted  
- [ ] Reorg policy written for credits and listings  

## 9. Related notes

- [001 — Wallet application threat model](001-wallet-app-threat-model.md)
- [003 — Bitcoin app backend hardening](003-bitcoin-app-backend-hardening.md)

## References

- [ordinals.com documentation](https://docs.ordinals.com/)
- [BIP 174 — PSBT](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)
- [BIP 370 — PSBT v2](https://github.com/bitcoin/bips/blob/master/bip-0370.mediawiki)
- Project-specific indexer docs (pin commit/version in any engagement SOW)
