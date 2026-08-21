# Silent Payment destination binding

**GreyBound Research · 005**  
**Status:** living note  
**Applies to:** hardware wallets, companion apps, and any signer that builds or approves BIP-352 Silent Payment outputs (and similar host-assisted payment-code destinations)

---

## 1. Intent

This note is the Bitcoin Silent Payment instance of [001 §5.5](001-wallet-app-threat-model.md): preview → sign is the critical boundary. [004](004-walletconnect-confirm-binding.md) is the EVM-session form. Same rule: the object signed must be the payment the user approved.

On 17 August 2026, Shift Crypto AG (BitBox) published the Dixence firmware update (9.26.5) and disclosed a Silent Payment implementation issue found during internal audits. Official framing: while creating a transaction to a Silent Payment address, a malicious host could cause funds to land on an unintended payment address. Direct theft of funds was not possible via this issue (vendor wording). Recovery would require cooperation between attacker and recipient — lock / ransom class, not spend-to-attacker.

This note extracts the engineering property. It is not a vendor autopsy, not a loss tally, and not a claim that self-custody failed. Dixence also covered other firmware issues with different affected editions and preconditions; those are out of scope here except where they remind reviewers to read “who is affected” carefully.

Primary source: [BitBox 08.2026 Dixence update](https://blog.bitbox.swiss/en/bitbox-08-2026-dixence-update/) (17 Aug 2026). The vendor reported no failed Silent Payments from users and no reason to believe this issue was exploited.

## 2. Why Silent Payments change the binding problem

On a normal send, the host proposes a `scriptPubKey` the user can (in principle) verify on-device against an address string. BIP-352 Silent Payments are different: the spendable output is a taproot output derived from the recipient’s Silent Payment address (scan and spend public keys) and a shared secret that uses the sender’s input private keys. The device is not only a signer; it participates in constructing the destination.

That makes host honesty about “where this goes” harder to check with a naive address reprint. It does not relax the rule. Privacy features still need fail-closed destination binding. The host is not the source of truth for the output that will be committed on-chain.

BitBox’s earlier public SP writeups ([Understanding Silent Payments — Part Two](https://blog.bitbox.swiss/en/understanding-silent-payments-part-two/)) describe Discrete Logarithm Equality (DLEQ) proofs so the *host* can check that the *device* used consistent key material when building the output (anti-klepto class). That is complementary. It is not destination binding: it does not prove that the output follows from the Silent Payment address the user approved on the trusted display. Do not treat DLEQ (or similar proofs) as a substitute for that bind.

## 3. What the disclosure stated (and what it did not)

Dixence’s TL;DR names **two severe** firmware issues (bootloader class, already fixed in Oeschinen 9.26.2; memory corruption on Multi, fixed in 9.26.5). The Silent Payment issue is described separately as **potentially severe**. Do not collapse those ratings.

| Claim | Per BitBox Dixence |
| --- | --- |
| Trigger class | User created a transaction to a Silent Payment address together with a malicious host |
| Impact class | Funds locked to an unintended payment address; potential ransom (attacker and recipient must cooperate to recover) |
| Direct theft | “There is no direct theft of funds possible” |
| Discovery | Internal audits (BitBox engineers; the post also describes frontier AI as part of that review program) |
| Exploitation | No reports of failed Silent Payments from users; no reason to believe it was exploited |
| Fix | Firmware 9.26.5 (Dixence) |
| SP-affected range (vendor) | BitBox02 and BitBox02 Nova, firmware **9.21.0 through 9.26.4**, for the SP send path above (`may be affected`) |

What the disclosure did not give (at publication of Dixence): a public root-cause writeup, PoC, or CVE-style technical appendix. Do not invent one in reviews or marketing. Cite the blog; treat missing mechanics as unknown until BitBox (or a coordinated writeup) publishes them.

## 4. Scope discipline

Dixence bundled more than Silent Payments. Memory-corruption and bootloader issues in the same post have different edition and setup preconditions:

| Issue | Who (vendor) | Precondition |
| --- | --- | --- |
| Bootloader (reclassified in Dixence; fixed in Oeschinen **9.26.2**) | BitBox02 on firmware through **9.26.1**. BitBox02 Nova **not affected** | Malicious app + malicious firmware; user installs it and unlocks |
| Memory corruption | **Multi** edition of BitBox02 and BitBox02 Nova through **9.26.4**. Bitcoin-only **not affected** (code path absent) | No wallet set up yet; malicious host |
| Silent Payment send | BitBox02 **and** BitBox02 Nova, **9.21.0–9.26.4** | Transaction to a Silent Payment address with a malicious host |

Silent Payment send is a Bitcoin feature. Do not collapse “Bitcoin-only is unaffected for the Multi memory issue” into “Bitcoin-only is out of scope for SP binding.”

Review habit: for every advisory, extract a matrix — edition, firmware range, user action required, host assumption — before drawing product lessons.

## 5. Required property

**Commitment:** the Silent Payment address (or equivalent payment code) shown and approved on the trusted display is the sole authority for deriving the BIP-352 output(s) that appear in the transaction the device signs. If the host supplies a different destination, different scan/spend keys, or a precomputed `scriptPubKey` that does not follow from that approval, the device refuses.

| Check | Expectation |
| --- | --- |
| On-device approval | Full Silent Payment address (or unambiguous truncated form with collision-resistant UX), network, amount, fee |
| Output construction | Derived on-device (or re-derived and verified on-device) from approved SP address + local spend policy |
| Host-provided `scriptPubKey` | Never trusted as authority; at most a hint that must match device derivation |
| Proofs (DLEQ / anti-klepto-class) | Optional integrity aid between host and device; not a substitute for binding approval → output |
| Mismatch / incomplete verify | Fail closed: no signature |

Same object identity rule as [001 §5.5](001-wallet-app-threat-model.md) and [101](101-psbt-mistakes.md): preview is not a commitment unless it is bound to the bytes that will be signed.

## 6. What to check in review

Trace one Silent Payment send end-to-end: companion app → device confirm → output derivation → sighash → signed tx.

1. What does the device display as the destination — raw SP address, label only, or a host-rendered summary?
2. Where is the BIP-352 output `scriptPubKey` computed? Device, host, or both with a proof?
3. If the host computes or suggests the output, does the device independently re-derive and compare before sign?
4. Can the host change the SP address after the user has confirmed on-device?
5. Can the host substitute a plain address / alternate output while the UI still says “Silent Payment to …”?
6. Are multi-output and change paths covered by the same binding (not only the SP output line)?
7. On verify failure, is sign impossible, or does the flow continue?
8. Are Bitcoin-only and multi-coin builds actually sharing the same SP code path for this check?
9. Is there a golden-vector test: fixed seed + fixed SP address + fixed inputs → fixed output script; mutation of host destination must refuse?

Hostile-host is the default assumption for hardware wallet review. “User already unlocked the device” does not make the host trusted for destination.

## 7. Hardening direction

- Treat Silent Payment send as a first-class policy path, not a stringly-typed address that falls through to “trust host `scriptPubKey`.”
- Keep derivation and binding on the side that holds the spend keys, with explicit on-device confirmation of the SP address.
- Fail closed on any proof, parse, or re-derive mismatch.
- Document edition differences; never assume “Bitcoin-only” or “Nova” without checking the advisory matrix.
- Add CI fixtures for SP sends the same way [004](004-walletconnect-confirm-binding.md) requires fixtures for transfer / approve: sheet (or device screen) text + committed output.

## 8. Control catalog

| ID | Control |
| --- | --- |
| S1 | On-device confirm of Silent Payment address (network-bound) |
| S2 | BIP-352 output derived or re-verified on-device from that approval |
| S3 | Host `scriptPubKey` never authoritative |
| S4 | Fail closed on derive / proof / display mismatch |
| S5 | Change and other outputs bound under the same approval |
| S6 | Hostile-host tests: mutated SP address or substituted output → no sign |
| S7 | Edition/firmware matrix recorded per advisory; no blanket scope claims |

## 9. Self-assessment

- SP send path traced device-side (not only companion UI)
- Approval bound to SP address string the device showed
- Host cannot supply an alternate output that still signs
- Verify failure blocks signature
- Golden vectors + hostile-host mutation tests in CI
- Advisory scope read per edition (not inferred from sibling issues in the same post)

## 10. Related notes

- [001 — Wallet application threat model](001-wallet-app-threat-model.md) (§5.5 preview → sign)
- [004 — WalletConnect confirm binding](004-walletconnect-confirm-binding.md) (same commitment class, EVM session)
- [101 — PSBT mistakes](101-psbt-mistakes.md) (preview is not a commitment)

## References

- [BitBox 08.2026 Dixence update](https://blog.bitbox.swiss/en/bitbox-08-2026-dixence-update/) — Silent Payment lock of funds; firmware 9.26.5; affected range 9.21.0–9.26.4 for SP send with a malicious host
- [BitBox — Understanding Silent Payments, Part Two](https://blog.bitbox.swiss/en/understanding-silent-payments-part-two/) — device-built output + DLEQ so the host can verify consistent device key material
- [BIP 352 — Silent Payments](https://github.com/bitcoin/bips/blob/master/bip-0352.mediawiki)
- GreyBound Research 001 — preview → sign as the highest-value wallet boundary
