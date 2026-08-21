# Classic k-of-n (P2WSH)

**GreyBound Research · 103**

---

## Why this note exists

Most “we built multisig” work fails before the first signature. The script is wrong, the backup cannot restore it, or the host is treated as the source of truth for the address. Coins then land on a `bc1q` nobody’s descriptor produces.

This is the builder contract for **classic** threshold: `OP_CHECKMULTISIG` behind **native SegWit P2WSH**, BIP48 account xpubs, BIP67, output descriptors, PSBT combine. It is what we kept after shipping and breaking a vault end-to-end (create → match → fund → sign → combine → restore).

It is not MuSig2. It is not Taproot `sortedmulti_a`. Those are different products. Mixing them in one path is how you get two addresses and one UTXO.

For preview → sign on the PSBT itself, see [101](101-psbt-mistakes.md). This note is the vault around that pipeline.

## 1. Pick one product

| | Classic P2WSH | MuSig2 (BIP327) | Taproot `tr(sortedmulti_a)` |
| --- | --- | --- | --- |
| On-chain | k signatures, n keys in the witness | One aggregated key / one signature | Threshold script, x-only keys, `CHECKSIGADD` |
| Setup | Share xpubs; no rounds | Interactive nonce + key agg | `tr()` descriptors, different PSBT fields |
| Sign | Independent; any k of n; combine later | Coordinated rounds; nonce reuse leaks the key | Still a script path, not key agg |
| Recovery | Full descriptor + k seeds | Agg key + participant state | Different backup than `wsh(sortedmulti)` |

Ship classic when the org wants: each person keeps a seed, they share pubkeys, they sign on their own devices, threshold then broadcast. That is the 2-of-3 / 3-of-5 treasury people already run in Sparrow / Coldcard / Caravan.

Do not reuse BIP84/86 singlesig keys as “the vault share.” Do not implement BIP45. BIP87 (`m/87'`) is cleaner on paper and weaker on hardware interop today. Default: BIP48 P2WSH `m/48'/0'/account'/2'` (testnet coin type `1'`).

**Architecture, not copies.** 2-of-3 only helps if the three seeds have independent entropy. Three keys born on the same weak generator are one bug. Threshold does not invent randomness.

## 2. The object you persist is the descriptor

A single P2WSH from three static pubkeys is not a wallet. There is no change chain, no gap scan, no restore.

Canonical 2-of-3:

```text
wsh(sortedmulti(2,
  [FPR0/48h/0h/0h/2h]XPUB0/<0;1>/*,
  [FPR1/48h/0h/0h/2h]XPUB1/<0;1>/*,
  [FPR2/48h/0h/0h/2h]XPUB2/<0;1>/*
))#CHECKSUM
```

Expand to two concrete descriptors (`/0/*` receive, `/1/*` change). Gap limit 20 on **both**. Backup is the **checksummed descriptor(s)** plus each seed. Lose one xpub and you cannot restore. Seed words alone are not a vault backup.

Always append BIP380 `#` + 8 characters. Reject checksum-less import in production. Golden-test checksums against Bitcoin Core `getdescriptorinfo`. Implementation drift here is permanent unspendability.

`sortedmulti` sorts **derived compressed child pubkeys at each index**, not the xpub strings, not mixed-case hex. `multi()` (caller key order) is forbidden: two coordinators produce two addresses.

Same child index on every cosigner (lockstep). Alice’s `/0/3` mixed with Bob’s `/0/7` is a script nobody backed up.

## 3. Ceremony: the collector is a bulletin

Naive flow: a web page computes the address; everyone funds what the page shows.

That is how an xpub swap steals or ransoms the treasury (attacker becomes a required cosigner, or the address is attacker-only).

Required flow:

1. Each party exports **fingerprint + origin + BIP48 xpub** (not a raw pubkey, not an xprv).
2. One person assembles k-of-n and publishes the checksummed descriptor plus receive index 0.
3. **Every** signer independently builds the same descriptor, checksums it, and shows address 0.
4. Funding UI stays disabled until this device’s index-0 receive equals the published string.
5. After verify, freeze membership. Adding or replacing an xpub is a **new** wallet and a new ceremony.

Show fingerprint and full origin, not a truncated xpub. The page must never ask for seeds. It must never be a dApp that calls `signPsbt`. HTTPS, no mixed content, no analytics dump of xpubs.

Watch-only combiner is a valid role: build, combine, broadcast; never hold a seed. Combiner still re-derives the descriptor. It is not an oracle.

## 4. Spend: roles, not magic

k signatures existing does not send coins. The pipeline is:

```text
coin-select descriptor UTXOs
  → unsigned PSBT (witnessUtxo + witnessScript + BIP32_DERIVATION)
  → k local signs (each device verifies outputs, then signInput)
  → combine the same template
  → finalize (NULLDUMMY)
  → extract → broadcast
```

Know the role: creator, updater, signer, combiner, finalizer. Incomplete extract hard-fails.

Unsigned PSBT per owned input must carry:

- `witnessUtxo` (the P2WSH output)
- `witnessScript` (the CHECKMULTISIG script)
- `BIP32_DERIVATION` for keys this device can sign
- sighash `SIGHASH_ALL` only on this wallet type

`witnessScript` must SHA256-bind to the `witnessUtxo` program. Unbound scriptCode is not a spend of that UTXO; it is still a signing oracle. Sign the **exact** derived witnessScript. Do not treat “our pubkey appears in this script” as ownership (`Buffer.includes` matches a foreign script that merely embeds the key).

This device’s fingerprint must appear in the descriptor before it will sign.

Combine only partials of the **same** unsigned template. Two signers on different destinations is not a threshold; it is garbage. Auto-broadcast from the collector when k blobs land lets the page substitute a PSBT. Finalize in a signer that already verified, or a watch-only combiner that re-checks the extracted tx against the original intent.

Let bitcoinjs / Core finalizers emit the extra dummy `OP_CHECKMULTISIG` pops. Hand-rolled finalizers that omit it never relay (`NULLDUMMY`). Do not assemble the script by concatenating opcodes unless you have a Core `testmempoolaccept` vector.

Fee-bump (RBF/CPFP) of a partially signed vault is not a 1-of-k spend. You need k new signatures on the new template.

## 5. Change is the vault

Every spend’s change output must match the **internal** `wsh(sortedmulti(…/1/*))` at the next unused index. Never send change to a personal singlesig address “for convenience.” That is a one-shot drain and a privacy break.

The home/balance view must show receive **and** change after a spend that created change. Scanning only `/0/*` makes restored wallets look empty.

Payment destination must not be the change address. Amount units (BTC vs sats) are part of the sheet; unit confusion on a treasury send is the same class as [101](101-psbt-mistakes.md) §3.

## 6. What must not share the singlesig path

This keyring is not a connected-site account. Provider `signPsbt` / `pushPsbt` must refuse it. Reusing the dApp approval sheet without the address-match and change checks is how a website spends the treasury.

Do not overload the HD singlesig keyring with BIP48 children. New type: local BIP48 signer **plus** imported cosigner xpubs (watch). Existing watch-address keyrings are singlesig-shaped; watching one `bc1q` is not k-of-n.

Inscription / runes / UTXO-picking tools built for singlesig will select the wrong sat. Disable them on this type until they speak the descriptor.

## 7. Limits and rejects (these are loss bugs)

| Rule | Why |
| --- | --- |
| Compressed 33-byte keys only | BIP67: uncompressed = incompatible implementations |
| k ≥ 2, k ≤ n, n ≥ 2 | k=1 is singlesig with extra steps; k=0 / k>n is unrecoverable policy |
| No duplicate xpubs or pubkeys | 2-of-3 with two copies of Alice is 1-of-2 |
| n cap: UX ≤ 7; consensus-standard P2WSH ≤ 20, P2SH ≤ 15, bare ≤ 3 | Over-size scripts validate and never relay |
| Native `wsh` (`bc1q`), not default `sh(sortedmulti)` (`3…`) | Hardware showing nested while software funds native (or reverse) loses deposits |
| `xpub` / `tpub` only; reject SLIP-132 `ypub`/`zpub`/`Ypub`/`Zpub` | Version byte lies about script type and network |
| Never mix `xpub` and `tpub`; coin type in origin matches network | Mainnet keys on testnet HRPs (and the reverse) |
| No xprv / WIF in the descriptor or export of this type | `/0/*` and `/1/*` are unhardened: leaked child priv + account xpub = the vault share |
| No `SIGHASH_NONE` / open `ANYONECANPAY` | Post-sign surgery on a treasury payment |

## 8. Tests before UI, signet before mainnet

If these are missing, the screens do not matter.

1. BIP67 vector: same witness script regardless of input key order.
2. 2-of-3 BIP48 xpubs → descriptor checksum matches Core `getdescriptorinfo`.
3. Address index 0 receive and index 0 change; Sparrow/Core import of the same descriptor yields the same `bc1q`.
4. Sign 2-of-3: two independent signers, combine, `testmempoolaccept` true.
5. Negatives: swapped xpub → address-match gate fires; sign refused if local fingerprint absent; `SIGHASH_NONE` refused; mixed `xpub`/`tpub` refused; duplicate keys refused; change to a singlesig path refused.

Mutation test from [101](101-psbt-mistakes.md): flip one output script after confirm; signer refuses.

## 9. Hardening checklist

- [ ] Product is classic `wsh(sortedmulti)` or you documented a different one (MuSig2 / `tr(sortedmulti_a)`) and did not mix paths
- [ ] BIP48 account xpubs with fingerprint + origin; no BIP84 reuse
- [ ] Checksummed descriptor persisted; receive **and** change; gap 20 both
- [ ] Every signer re-derives; fund disabled until address 0 matches
- [ ] Membership frozen after verify
- [ ] Change is internal `/1/*`; balances scan both chains
- [ ] Local unsigned PSBT; `witnessUtxo` + `witnessScript` bound; `SIGHASH_ALL`
- [ ] Combine same template; finalize with NULLDUMMY; no collector auto-broadcast
- [ ] Treasury keyring cannot be a dApp account
- [ ] Compressed keys; k≥2; no dupes; network bind; no SLIP-132 mix
- [ ] Core checksum + BIP67 + 2-of-3 extract in CI; signet round-trip before mainnet

## Related notes

- [001](001-wallet-app-threat-model.md) — §5.4 build / §5.5 preview → sign
- [101](101-psbt-mistakes.md) — PSBT preview, sighash, change, combine vs finalize
- [102](102-dlc-adaptor-intuition.md) — adaptor / Taproot contracts (not this vault)
- [005](005-silent-payment-destination-binding.md) — another case where the host must not author the output

## References

- [BIP 48](https://github.com/bitcoin/bips/blob/master/bip-0048.mediawiki) — account paths for multisig script types (`2'` = P2WSH)
- [BIP 67](https://github.com/bitcoin/bips/blob/master/bip-0067.mediawiki) — lexicographic sort of compressed pubkeys
- [BIP 174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) — PSBT
- [BIP 380](https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki) / [BIP 382](https://github.com/bitcoin/bips/blob/master/bip-0382.mediawiki) / [BIP 383](https://github.com/bitcoin/bips/blob/master/bip-0383.mediawiki) — descriptors, `wsh`, `sortedmulti`
- [BIP 129](https://github.com/bitcoin/bips/blob/master/bip-0129.mediawiki) — BSMS spirit: persist the descriptor; verify receive and change
- Bitcoin Core `getdescriptorinfo` — checksum and address oracle for tests
