# PSBT mistakes Bitcoin developers keep making

**GreyBound Research · 101**  
**Code:** [psbt-signer](https://github.com/Grey-Bound/opensource/tree/main/psbt-signer)

---

## Why this note exists

Most wallet and marketplace bugs we see are not “ECDSA failed.” They are PSBT pipeline mistakes: the UI describes one payment, the signer commits to another, or a helper service authors fields the client never re-checks.

This is a practical list of misuse patterns at moderate depth—enough to catch real errors, not a full BIP-174 commentary. For the security framing around preview↔sign, see [001](001-wallet-app-threat-model.md).

## 1. Preview is not a commitment

**Misuse:** Render a friendly summary from JSON (`to`, `amount`, `fee`) while `signPsbt(psbtBase64)` consumes a different object that was updated on another port or after the click.

**Correct:** Bind approval to a hash of the exact PSBT (or the extracted unsigned tx) that will be signed. Recompute the summary from that same blob. If the blob changes, approval is void.

**Check:** Mutation test in CI—flip one output script after “confirm”; signer must refuse or require a new approval.

## 2. Sighash flags treated as decoration

**Misuse:** Accept `SIGHASH_NONE`, `SINGLE`, or `ANYONECANPAY` on paths that the UI presents as a normal fixed payment. Users approve “pay 0.01 to bc1q…” while the sighash allows input/output surgery later.

**Correct:** Default policy: only `SIGHASH_ALL` (and the Taproot default equivalent) unless the product explicitly supports advanced contracts and shows the flag in the preview.

**Check:** Reject unexpected sighash in the signer before prompting for keys.

## 3. Change outputs omitted or mislabeled

**Misuse:** Build PSBTs that send change to an attacker-controlled path, or hide change so “send max” looks like a single output when fee math left a remainder elsewhere.

**Correct:** Every non-payment output is visible (address + amount + “change” / “fee” / “postage”). Descriptor wallets should verify change against their own policy.

## 4. Server-authored PSBTs signed blindly

**Misuse:** Marketplace or minter returns a full PSBT; wallet shows one line of copy and signs. Common in inscription/BRC-20 flows.

**Correct:** Rebuild or independently decode: inputs owned by the user, output scripts and amounts, fee rate, inscription/postage lines. Treat the server as hostile ([002](002-metaprotocol-indexer-trust.md)).

**Practice:** [psbt-signer](https://github.com/Grey-Bound/opensource/tree/main/psbt-signer) can decode Taproot leaves before signing; product UX should not skip that step either.

## 5. Finalization vs signing confusion

**Misuse:** Broadcast a PSBT that still needs another signature, or “finalize” client-side incorrectly and produce an invalid tx. Or assume `extractTransaction` is safe without checking complete signatures.

**Correct:** Know your role: creator, updater, signer, finalizer. Multisig and DLC flows need an explicit state. Incomplete extract should hard-fail.

## 6. Locktime / sequence ignored in UX

**Misuse:** Refund or CSV/CLTV paths signed without showing `nLockTime` / `nSequence`. Users think they can broadcast immediately; nodes reject or funds stay stuck until height/time.

**Correct:** Preview must show absolute/relative lock constraints. Signers should warn when wall-clock or tip makes the tx non-final.

## 7. Network and address HRP mixups

**Misuse:** Mainnet keys used with testnet/signet/FB HRPs in the same PSBT, or decoding an address for the wrong chain and stuffing the scriptPubKey anyway.

**Correct:** Network is part of policy. Fail closed on HRP mismatch. Multi-chain tools (the lab signer supports several) must require an explicit network selection before interpret/sign.

## 8. BIP32 paths and proprietary fields trusted from strangers

**Misuse:** Trust `bip32_derivation` or proprietary PSBT keys from a counterparty to decide which key to use, or to display “account names.”

**Correct:** Derivation paths are hints for *your* wallet software against *your* seed. Counterparty-provided paths never authorize a spend by themselves.

## 9. Fee rate as an unreviewed input

**Misuse:** Helper API sets fee; client never shows sat/vB or absolute fee; RBF signaling unclear.

**Correct:** Show absolute fee and feerate. Cap absurd fees. If RBF is enabled, say so.

## 10. Logging PSBTs and keys

**Misuse:** Debug logs or support uploads contain full PSBT hex (sometimes with enough structure to aid theft) or worse, WIF.

**Correct:** Redact. If you must log, log txid, fingerprints, output addresses—not seeds, xprv, or raw private material.

## Minimal engineering checklist

- [ ] Preview fields derived from the to-be-signed PSBT bytes  
- [ ] Sighash policy enforced  
- [ ] All outputs listed; change verified to local policy  
- [ ] Server PSBTs decoded/rebuilt locally  
- [ ] Locktime/sequence shown when relevant  
- [ ] Network/HRP explicit  
- [ ] No secrets in logs  

## Related notes

- [001](001-wallet-app-threat-model.md) — preview → sign
- [005](005-silent-payment-destination-binding.md) — BIP-352 output must follow the approved Silent Payment address, not a host `scriptPubKey`
- [103](103-classic-p2wsh-multisig.md) — classic k-of-n around this PSBT pipeline

## References

- [BIP 174 — PSBT](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)
- [BIP 370 — PSBT v2](https://github.com/bitcoin/bips/blob/master/bip-0370.mediawiki)
- [BIP 371 — Taproot PSBT fields](https://github.com/bitcoin/bips/blob/master/bip-0371.mediawiki)
