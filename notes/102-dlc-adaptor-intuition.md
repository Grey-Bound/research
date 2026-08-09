# Adaptor signatures and Taproot contracts

**GreyBound Research · 102**  
**Code:** [taproot-dlc-lab](https://github.com/Grey-Bound/opensource/tree/main/taproot-dlc-lab) · [psbt-signer](https://github.com/Grey-Bound/opensource/tree/main/psbt-signer)

---

## Scope

This note explains BIP-340 **adaptor signatures** used with **Taproot** contract outputs in the Discreet Log Contract design space. There is no Bitcoin standard called “DLC v2.”

For PSBT handling mistakes that break these flows in practice, see [101](101-psbt-mistakes.md).

## Adaptor signatures in one page

Goal: tie two on-chain actions so that finishing one reveals a secret the counterparty needs for the other—without an escrow that holds withdraw keys.

With a BIP-340 adaptor construction:

1. The signer creates a **pre-signature** relative to adaptor point `T = t·G` (scalar `t` held by the secret-holder).
2. The pre-signature is not a valid on-chain Schnorr signature by itself.
3. **Complete** with `t` → a normal 64-byte Schnorr signature.
4. From pre-signature + completed signature, anyone can **extract** `t`.

If a party broadcasts a claim that required `t`, they publish `t`. That extractability is the atomic hinge—when the protocol actually uses it.

## What the lab implements

| Piece | Where | Trust hinge |
|-------|-------|-------------|
| Swap / loan-delivery builder | `dlc_builder/` | Claim completeness reveals `t`; NUMS internal key; script-path spends |
| Collateral builder | `lending_dlc_builder/` | Three-leaf tree; attestation mode matters |
| Offline PSBT tool | `psbt-signer/` (sibling) | Decode leaves before signing |

Older coordinator co-sign claim scripts (adaptor key on-script) are not shipped. If a product still needs an operator co-sign on every claim, say so; do not market it as purely cryptographic atomicity.

## Taproot output shape (adaptor construction)

Typical script paths in the lab builder:

- **Claim** — `<receiver_xonly> CHECKSIG` (often an ephemeral per-contract key)
- **Refund** — CLTV (or similar) + sender key after timeout

Internal key: NUMS (no known discrete log), so spends are expected via script paths.

Adaptor point `T` is kept off-chain; it does not need to be embedded in the address.

Correct PSBTs still matter: wrong control block, wrong leaf, or ignored locktime strands funds even when adaptor math is correct.

## Mistakes we see

- Labeling any HTLC or multisig escrow a “DLC”
- Holding `t` on a server and calling the system trustless
- Reusing claim keys across contracts
- Hiding refund timeouts from the user
- Blind-signing server-built PSBTs without decoding leaves
- Copying swap claim semantics onto lending/collateral trees without reading the attestation model

## Open code

- **taproot-dlc-lab** — builders, adaptor helpers, `PROTOCOL.md`
- **psbt-signer** — offline PSBT inspect/sign (sibling project)

```bash
cd taproot-dlc-lab && source .venv/bin/activate
export PYTHONPATH=.
python3 dlc_builder/test_roundtrip.py
python3 ../psbt-signer/signer.py
```

## References

- Lab `PROTOCOL.md`
- [BIP 340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki), [BIP 341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki), [BIP 342](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)
- Discreet Log Contracts (Dryja et al.) for the broader design space
