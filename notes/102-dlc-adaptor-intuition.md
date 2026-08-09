# DLC and adaptor-signature intuition

**GreyBound Research · 102**  
**Companion lab:** [taproot-dlc-lab](https://github.com/Grey-Bound/opensource/tree/main/taproot-dlc-lab)

---

## Intent

Discreet Log Contracts and Schnorr adaptor signatures show up in bridges, atomic swaps, and lending locks. Teams often ship something that *looks* like a DLC but still trusts a coordinator with a co-sign key—or builds HTLCs and calls them “atomic” without the same failure modes.

This note is the intuition layer. The lab contains the v2 builder and adaptor math; the full product (matching, APIs, monitoring) is intentionally not open-sourced ([WHAT_WE_RELEASE](https://github.com/Grey-Bound/opensource/blob/main/taproot-dlc-lab/WHAT_WE_RELEASE.md)).

## What problem adaptors solve

You want two on-chain effects tied together so that completing one reveals enough for the counterparty to complete the other—without an escrow that holds withdraw keys.

With **BIP-340 adaptor signatures** (lab: `dlc_v2_builder/adaptor_sig.py`):

1. Signer produces a **pre-signature** under adaptor point `T = t·G` (secret `t` held by the secret-holder).  
2. Pre-signature alone is not a valid on-chain Schnorr sig.  
3. **Complete** with `t` → valid 64-byte Schnorr signature.  
4. Anyone who sees pre-sig + completed sig can **extract** `t`.

So: broadcasting a claim that used `t` publishes `t` to the peer. That is the atomic hinge—if your protocol actually uses it.

## v1 co-sign vs v2 adaptor (why the lab separates them)

| | Coordinator co-sign (legacy v1) | Adaptor v2 (lab primary) |
|--|----------------------------------|---------------------------|
| Claim path | Often needs a second key the coordinator controls | Receiver key + adaptor completeness |
| Trust | Coordinator can stall or collude via co-sign | Atomicity from extractability of `t` |
| Address | May bake adaptor into script | v2 keeps `T` off-chain; NUMS internal key; script-path spend |

If your “DLC” still has an operator key that must co-sign every claim, you have an **ops dependency**, not full cryptographic atomicity. That can be an acceptable product choice—but label it honestly in reviews and UX.

## Taproot shape (v2 swap, simplified)

Lab builders construct a Taproot output with script paths roughly:

- **Claim** — `<receiver_xonly> CHECKSIG` (ephemeral claim key per swap)  
- **Refund** — CLTV + sender path after timeout  

Internal key is a **NUMS** point (no known discrete log) so spends go through script paths, not a surprising key-path unilateral exit.

Funding, claiming, and refunding still need correct PSBTs ([101](101-psbt-mistakes.md)): wrong control block, wrong leaf, or ignored locktime will strand funds even if adaptor math is perfect.

## Mental model for a two-chain swap

```text
  Chain A lock (Taproot DLC)          Chain B lock (Taproot DLC)
         │                                    │
         │     adaptor pre-sigs exchanged     │
         │◄──────────────────────────────────►│
         │                                    │
         │     claim with t on one chain      │
         │     reveals t ─────────────────────┤
         │                                    │ peer completes other claim
```

Details (who holds `t`, timeouts, how presigns are ordered) live in `PROTOCOL.md` in the lab. Read that before copying fragments into production.

## Common protocol mistakes

1. **Calling HTLC a DLC** — Hashlocks are fine tools; they are not adaptor DLCs. Different griefing and privacy properties.  
2. **Coordinator holds `t`** — Then atomicity is social. Prefer secret-holder = party who should reveal by claiming.  
3. **Reusable claim keys** — v2 lab uses ephemeral claim keys; reusing keys across swaps couples privacy and blast radius.  
4. **Ignoring refund timers** — UI must surface CLTV/CSV; signers must respect `nLockTime`.  
5. **Server builds PSBT, client blind-signs** — Same as marketplace PSBT risk; decode leaves before signing (`Signer/`).  
6. **Lending vs swap confusion** — Collateral 3-leaf contracts (`lending_dlc_builder`) have different attestations (oracle / FAL / fixed-term). Do not assume swap claim semantics.

## What to open-source vs keep private

From the NexumBit release policy (mirrored in the lab):

**Open:** builders, adaptor math, offline signer, protocol doc.  
**Private:** matching, loan monitor, API keys, frontend, deploy, live recovery kits with funds.

That split is deliberate. Cryptography libraries do not replace operational review of a bridge.

## How GreyBound uses this

- **Protocol implementation review** — Is your atomicity cryptographic or coordinator-gated? Are PSBT/locktime paths safe?  
- **Builder education** — Lab + this note.  
- **Not for sale as “download our bridge and go mainnet.”**

## Lab exercises

```bash
cd taproot-dlc-lab && source .venv/bin/activate
export PYTHONPATH=.
python3 dlc_v2_builder/test_roundtrip.py    # adaptor extract works
python3 dlc_v2_builder/example_swap.py      # descriptor build sketch
python3 Signer/signer.py                    # decode PSBT leaves before sign
```

## References

- Lab `PROTOCOL.md`  
- [BIP 340 — Schnorr](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)  
- [BIP 341 — Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)  
- [BIP 342 — Tapscript](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)  
- Discreet Log Contracts literature (Dryja et al.) for the broader design space  
