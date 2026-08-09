# Wallet application threat model

**GreyBound Research · 001**  
**Status:** living note  
**Applies to:** hot wallets, browser extensions, mobile wallets, desktop signers, and companion apps that construct or approve Bitcoin transactions

---

## 1. Intent

This note is the threat model Greybound uses when reviewing Bitcoin wallet and signing applications that will hold or move mainnet value.

It is written for engineering leads and security reviewers who need a shared map: what is in scope, what must be true for funds to stay safe, and where production wallets actually fail. It is not a CVE list, not a hardware silicon assessment, and not a substitute for reading the code under review.

A usable threat model does three things:

1. Names **assets** worth attacking.
2. Names **trust boundaries** where those assets change hands between components.
3. Ties failures to **fund loss, key loss, or mass user impact**—not abstract “best practices.”

Everything below is organized that way.

## 2. System under review

We treat a modern wallet as a pipeline, not a single binary:

```text
  entropy → seed/keystore → derivation → address/UTXO view
       → tx/PSBT build → preview/approve → sign → broadcast
                              ↑                ↑
                     indexer / APIs      update channel
                     dapp / deep link    platform OS
```

Any stage can be honest while another is hostile. Reviews that only inspect “is the seed encrypted at rest?” miss the stages where most real theft happens: build, preview, and approve.

### In scope

- Key generation, import, backup, and export
- Derivation paths, networks, descriptors, script policies
- Address display and QR handling
- Coin selection, fee estimation, RBF/CPFP controls (when present)
- PSBT and sighash construction; multisig / miniscript presentation
- Approval UX (biometrics, sessions, connect flows)
- Extension / dapp / deep-link request handling
- Companion HTTP APIs used during build or display
- Logging, crash reporting, clipboard, screenshots, backups
- Release signing and in-app update mechanisms

### Out of scope (unless contracted)

- Breaking secure elements or Coldcard/Trezor firmware as a primary objective
- Full OS malware development
- Consensus bugs in Bitcoin Core
- Economic attacks that require hashrate or fee-market dominance alone

Hardware wallets still appear in the model: as a **boundary** that moves key material off the host. The host UI, descriptor setup, and companion software remain fully in scope.

## 3. Assets

| Asset | Impact if lost or forged | Typical location |
|--------|---------------------------|------------------|
| BIP39 mnemonic / seed | Total fund loss, indefinite | Secure enclave, Keystore, encrypted DB, sometimes plaintext backup |
| Root / account xprv | Same as seed for that account tree | Same |
| Descriptor + private material | Wrong script or leaked keys | Config + keystore |
| Active PSBT / unsigned tx | Theft if mutated post-preview | Memory, temp files, extension message bus |
| Wallet unlock secret | Gate to the above | Biometrics, PIN, password |
| Backup blob / cloud export | Often equivalent to seed | iCloud/Google, files, QR archive |
| Release signing key / update trust | Silent future theft at scale | CI, HSM, store listing |
| Session / dapp permission | Ambient spend or message sign | Local storage |

Prioritize assets by **blast radius × exposure**. A weak vault password on a widely distributed extension beats an obscure edge-case UI glitch.

## 4. Adversaries

| Adversary | Capabilities | Realistic goals |
|-----------|--------------|-----------------|
| Malicious website / dapp | Origin-scoped script, deep links, clipboard bait | Blind sign, address swap, infinite approval |
| Malicious extension / overlapping host permissions | Message interception, DOM overlay | PSBT mutation, seed phishing overlays |
| Compromised companion API | Lie about balances, fees, inscriptions, templates | Induce wrong sign; phishing via “support” payloads |
| Local malware / jailbreak | Memory read, hooking, accessibility abuse | Seed exfiltration, overlay approve |
| Supply chain (npm/CocoaPods/CI) | Ship hostile update | Mass key drain after trust established |
| Physical opportunistic | Unlocked device, 15 minutes | Export, tx to attacker, disable lock |
| Malicious insider (small teams) | Release keys, backend admin | Targeted or broad theft |

Do not waste review time on nation-state side channels before you have checked Math.random-class secret generation and preview≠sign.

## 5. Trust boundaries (detailed)

### 5.1 Entropy → seed

**Required property:** every secret that protects value is sampled from a CSPRNG appropriate to the platform (`crypto.getRandomValues`, SecureRandom, `getrandom`, etc.).

**Common breaks:**

- `Math.random`, timestamp, or device-derived low entropy for “temporary” encryption keys that wrap the seed
- Seed shown in UI that is screenshot-able, readable by accessibility APIs, or logged
- Mnemonic import that checksum-ignores or auto-corrects words
- Deterministic “random” in tests accidentally left in production builds

**Evidence in review:** grep and data-flow for all `random` / password / key-wrap sites; prove production builds cannot hit weak paths; confirm wipe of seed from memory/UI buffers after use where the platform allows.

### 5.2 Seed → addresses (derivation & policy)

**Required property:** scriptPubKey and derivation are pure functions of explicit network + policy. Untrusted input cannot silently change coin type, account, or script type.

**Common breaks:**

- Mainnet/testnet key reuse with only a UI badge changing
- Hardcoded account `0` while UI suggests multiple accounts
- Descriptor string concatenated from server or QR without checksum verify (`descsum`)
- Multisig policies displayed as “2-of-3” while the descriptor is something else

**Evidence:** unit tests for path/network; descriptor parse + checksum; fixtures for UI vs derived address vectors.

### 5.3 Chain view → user belief

Wallets show balances and UTXOs from electrum/esplora/indexer/custom APIs. That data is **not** consensus.

**Required property:** display can be wrong without forcing a wrong signature. Signing inputs must be verified against local policy (script, amount, outpoint) to the extent the wallet claims to be non-custodial.

**Common breaks:**

- Server-chosen inputs signed without showing full set
- “Send max” using server fee that leaves dust the user did not expect
- Inscription/rare-sat metadata trusted into coin selection without local tags

### 5.4 Intent → PSBT (build)

**Required property:** the PSBT (or legacy tx) is fully determined by user intent + local policy. Helper APIs may suggest; they must not authoritatively define outputs the user did not approve.

**Common breaks:**

- Change output omitted or pointed at an attacker path
- Extra output inserted after UI confirm
- Amount unit confusion (sats vs BTC vs localized decimal)
- Locktime/sequence set in ways that enable fee theft or pinning the user did not accept

**Evidence:** golden-vector tests: given intent JSON, assert PSBT global/input/output fields. Mutation tests: any post-preview change invalidates approval.

### 5.5 Preview → sign (the critical boundary)

This is the highest-value boundary in almost every wallet review.

**Required property:** cryptographic commitment. The digest signed must be exactly the transaction the preview described—every output address and amount, fees, and relevant script path.

**Common breaks:**

- Preview rendered from a friendly JSON while signer consumes a different PSBT object
- Race: PSBT updated on another thread/extension port after approve click
- Sighash flags not shown (`NONE`, `SINGLE`, `ANYONECANPAY`) on non-standard paths
- Partially signed multisig where the user thinks they signed “the payment” but signed a different template

**Evidence:** single object identity from preview through sign; hash of canonical tx hex shown or logged at debug in test builds; rejection of PSBTs with unexpected sighash.

### 5.6 Approvals and sessions

**Required property:** value-moving actions require a fresh, complete approval. Ambiguous biometrics (“confirm”) are not sufficient for arbitrary dapp payloads.

**Common breaks:**

- Connect sessions that allow signing forever
- Notification or Autofill-style confirms
- “Remember this site” that includes transaction sign
- WalletConnect-style sessions with broad methods enabled by default

### 5.7 Export, backup, support

**Required property:** export is explicit, authenticated, and hard to do accidentally. Support tooling cannot pull seeds remotely.

**Common breaks:**

- Seed in Sentry/Bugsnag payloads
- Verbose debug logs in production
- Backup to cloud without strong KDF (low iteration PBKDF2, unsalted)
- “Restore from clipboard” without warning

### 5.8 Update and supply chain

**Required property:** only the publisher’s key can change code that can touch keys. Users can detect rollback where the platform allows.

**Common breaks:**

- HTTP update endpoints
- Shared debug/release signing keys
- Auto-updating WebView content that includes signing logic
- Dependency hijack on packages imported by the signer path

## 6. Attack scenarios (classes)

These are scenario classes used in review planning. They are deliberately generic.

### A — Weak wrap key

Attacker obtains encrypted vault blob (backup, device extract, synced storage). Wrap key was generated with non-CSPRNG or low-entropy password without adequate KDF. Offline brute force yields seed.

### B — Address swap on hostile page

User pastes or scans an address. Malicious page or extension replaces clipboard/QR decode result after display, or overlays a different address. Preview is skipped or shows truncated address that collides visually.

### C — PSBT mutation

Dapp proposes PSBT. Wallet shows summary. Before sign, extension bus delivers a mutated PSBT with altered output. Signer does not re-bind approval to PSBT hash.

### D — Indexer-induced mis-spend

Hostile or buggy indexer marks an inscribed UTXO as ordinary. Coin selection spends it as fee. User approved “send 0.001 BTC” without inscription warning.

### E — Malicious update

CI signing key leaks. Attacker ships a build that exfiltrates seed on next unlock. Users update because the app is “from the same store listing.”

### F — Ambient dapp session

User connected wallet last month. Site calls sign methods without clear per-tx consent. Policy allowed it at connect time.

A review should say which of these are in scope for the product’s architecture and what control closes each one.

## 7. Control catalog

Minimum controls we expect on a mainnet wallet. Not all apply to every architecture; justify skips in writing.

| ID | Control | Boundary |
|----|---------|----------|
| C1 | CSPRNG-only secret generation; no dual path in prod builds | Entropy |
| C2 | Memory hygiene for seed material after use | Keystore |
| C3 | Descriptor checksum + network binding | Policy |
| C4 | Preview fields derived from the same PSBT object that is signed | Preview/sign |
| C5 | Explicit sighash policy; reject unexpected flags | Sign |
| C6 | Per-transaction approval for value movement; no ambient sign | Approve |
| C7 | Origin binding for external requests | Dapp |
| C8 | Treat indexer data as untrusted for signing decisions | Chain view |
| C9 | Production logging redaction; no seed/PSBT/xprv | Export |
| C10 | Strong KDF for password-wrapped backups (memory-hard where possible) | Backup |
| C11 | Signed updates; separate debug channels | Release |
| C12 | Dependency pin + review for signer-path packages | Supply chain |
| C13 | Change output always visible when not send-max-to-self by policy | Build |
| C14 | Test vectors: intent → PSBT → signed tx hex | Eng process |

## 8. How Greybound reviews this in practice

1. **Architecture interview** — draw the pipeline with the team; mark trust boundaries.
2. **Secret archaeology** — find every RNG, KDF, keystore, backup, and log site.
3. **Signing path trace** — one transaction type end-to-end from UI intent to signature; prove object identity.
4. **Hostile input pass** — deep links, PSBT fixtures, malformed descriptors, oversized messages.
5. **Release and update pass** — how a build becomes what users run.
6. **Report** — ranked findings, reproduction notes for serious issues, fix direction in their repo, executive summary.

We care about proof. “We use industry best practices” without a path from intent to sighash is not a finding closure.

## 9. Self-assessment (engineering teams)

Score each item: **Done / Partial / Missing**.

- [ ] Documented key hierarchy and network separation  
- [ ] Weak RNG hunt completed on production entry points  
- [ ] Backup/export threat model written and tested  
- [ ] Preview↔sign binding tested with mutation cases  
- [ ] Dapp/session permissions matrix exists  
- [ ] Indexer failure mode documented (lie, lag, reorg)  
- [ ] Production log samples redacted and reviewed  
- [ ] Release signing and update story written  
- [ ] One full golden-vector test per major tx type  

Below ~70% Done is not a branding problem; it is a launch risk.

## 10. Related notes

- [002 — Metaprotocol and indexer trust](002-metaprotocol-indexer-trust.md) when the wallet surfaces inscriptions or metaprotocol balances  
- [003 — Bitcoin app backend hardening](003-bitcoin-app-backend-hardening.md) when a server can influence invoices, webhooks, or templates  

## References

- [BIP 174 — PSBT](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)
- [BIP 370 — PSBT version 2](https://github.com/bitcoin/bips/blob/master/bip-0370.mediawiki)
- [BIP 371 — PSBT taproot fields](https://github.com/bitcoin/bips/blob/master/bip-0371.mediawiki)
- [BIP 380–386 — Output script descriptors](https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki)
- [BIP 39 — Mnemonic code](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)
