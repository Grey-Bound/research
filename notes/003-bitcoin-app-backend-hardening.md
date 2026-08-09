# Bitcoin app backend hardening

**GreyBound Research · 003**  
**Status:** living note  
**Applies to:** APIs, indexers, payment processors, store dashboards, and any server that can create economic side effects beside a Bitcoin app

---

## 1. Intent

Non-custodial wallets still depend on servers: UTXO queries, fiat ramps, invoice engines, marketplaces, push, and “helpful” PSBT builders. Custodial and hybrid products depend on them even more.

This note is the backend and operations threat model Greybound uses in production readiness and infrastructure work. It pairs with [001](001-wallet-app-threat-model.md) (client) and [002](002-metaprotocol-indexer-trust.md) (opinion layer). Lightning-specialist protocol review is not the focus here.

## 2. What “backend” means here

Anything that is not the user’s signing device, including:

- REST/GraphQL/gRPC APIs (user, store, admin, machine)
- Webhook dispatchers and callback workers
- Indexer and chain-query services
- Hot wallet / payout workers (if present)
- CI/CD and secret stores that can change the above
- Object storage for backups, KYC, or export dumps

If it can cause a merchant to ship goods, a payout to leave, or a client to construct a transaction, it is in scope.

## 3. Assets

| Asset | Blast radius |
|--------|----------------|
| Store / tenant API keys | Impersonate merchant; pull data; trigger payouts |
| Admin session | Cross-tenant god mode |
| Hot wallet descriptors / xprv / RPC auth | Direct fund loss |
| Webhook signing secret | Forge payment events |
| Invoice private metadata | Fraud, privacy loss |
| CI deploy credentials | Silent policy change |
| Database snapshots | Mass PII + payment history |

## 4. Adversaries

- External API attacker (stolen key, auth bug, IDOR)
- Merchant A attacking merchant B (multi-tenant)
- Hostile webhook target (SSRF, slowloris against workers)
- Compromised dependency or base image
- Malicious or phished employee with cloud admin
- Chain reorg / indexer desync as an reliability adversary (credits wrong)

## 5. Boundaries and controls

### 5.1 Authentication vs authorization

**Required property:** proving who you are is not the same as what you may do to which tenant.

**Concrete expectations:**

- Store-scoped API keys with explicit permission bits (invoice create ≠ payout submit ≠ settings write)
- User sessions cannot call admin routes by “forgetting” a check
- Machine keys distinct from human SSO
- Rotation and revocation without redeploying the world

**Failures:** global keys in mobile apps; “isAuthenticated” without `storeId` match; support impersonation without TTL or audit.

### 5.2 Multi-tenant object access

**Required property:** every fetch/update by id re-validates tenant ownership in the same transaction as the mutation.

**Failures:**

- `/invoices/{uuid}` returns any store’s invoice
- Cache key `invoice:{id}` without tenant prefix
- Background worker trusts payload `storeId` from the queue message forged earlier
- Elasticsearch/OpenSearch indexes without tenant filter defaults

**Evidence:** automated IDOR tests across two seeded stores; negative tests on workers.

### 5.3 Webhooks and outbound HTTP

**Required property:** merchant-controlled URLs cannot make your infrastructure reach internal targets, and payment events cannot be forged or replayed into double fulfillment.

**Controls:**

- SSRF allow/deny: block link-local, metadata IPs (`169.254.169.254`), RFC1918 unless explicitly opted in for self-host weirdness
- Fixed DNS resolve + pin to resolved address for the request (DNS rebinding aware)
- Timeouts, max response size, redirect limits
- HMAC signatures with timestamp skew window
- Idempotency keys for fulfillment side effects

**Failures:** `url = user_input` into bare `HttpClient.GetAsync`; retries that create two shipments; unsigned callbacks.

### 5.4 Invoice and payment state machines

**Required property:** a single explicit state machine. Terminal means terminal. Amount/address/asset frozen at issue (or versioned with a new id).

**States worth naming:** `draft → unpaid → paid_unconfirmed → paid → expired → invalid` (names vary). Document who can transition, and whether under/over payment creates a new state or a child invoice.

**Failures:**

- Paid invoice amount editable
- Expired invoice reopened by probing
- Zero-conf credit without matching risk policy
- Detection path and manual “mark paid” path with different checks

**Evidence:** transition table; property tests; audit log of transitions.

### 5.5 Chain connectors

**Required property:** RPC and indexer credentials are not on the public internet; spend authority is minimized and monitored.

**Controls:**

- bitcoind/electrs bound to localhost or private net; reverse proxy auth
- Separate watch-only vs spending roles
- Confirmation policy per asset/value band
- Reorg handling: revoke or flag credits when depth violated

**Failures:** RPC on `0.0.0.0` with cookie auth alone; same wallet for fees and customer withdrawals; credits at 0-conf for high value with no fraud controls.

### 5.6 Secrets and configuration

**Required property:** production secrets are not in git, world-readable images, or CI logs. Staging cannot mint production authority.

**Controls:** secret manager; short-lived CI OIDC; config hash recorded per deploy; panic if prod boots with staging keys.

### 5.7 Deploy and feature flags

**Required property:** changing spend policy is a production event—reviewed, logged, reversible.

**Failures:** remote flag disables signature verification on webhooks; “maintenance mode” that auto-approves payouts; one-click deploy from laptop without SSO.

## 6. Scenario classes

### B1 — Cross-tenant payout

Stolen or guessed invoice id + weak ACL → attacker watches merchant A’s invoices or redirects refunds.

### B2 — Webhook SSRF to metadata

Attacker sets webhook to cloud metadata, steals instance role, empties hot wallet bucket or redeploys.

### B3 — Forged callback

No HMAC. Attacker POSTs “paid” to merchant’s own system or to your internal fulfillment bus.

### B4 — Invoice bait-and-switch

Invoice shared with payer; merchant API changes amount downward after share; auto-fulfill still triggers.

### B5 — Reorg double credit

Ship goods at 1-conf; reorg; attacker reuses funds; no clawback process.

### B6 — CI as admin

Deploy token in GitHub Actions from fork PR; attacker ships image that exfils DB.

## 7. Control catalog

| ID | Control |
|----|---------|
| B1 | Store-scoped least-privilege API keys + rotation |
| B2 | Tenant check on every object access (incl. workers) |
| B3 | SSRF-safe webhook egress |
| B4 | Signed, timestamped, idempotent webhooks |
| B5 | Frozen invoice economic fields after issue |
| B6 | Explicit payment state machine + audit log |
| B7 | RPC/indexer network isolation |
| B8 | Confirmation + reorg policy documented and enforced |
| B9 | Secret manager; no secrets in images |
| B10 | Deploy provenance (who/what digest/config) |
| B11 | Rate limits on auth and payout routes |
| B12 | IDOR/regression suite in CI |

## 8. Review method

1. Inventory every route and worker that can move value or assert payment.
2. Build a tenant matrix (Store A/B keys) and run IDOR scripts.
3. Point webhooks at an internal capture service; try metadata and redirect tricks in a lab network.
4. Walk invoice states with concurrency (double pay, cancel storms).
5. Inspect RPC exposure and wallet roles on a staging clone of prod topology.
6. Read CI from the perspective of “can a PR steal prod?”.
7. Deliver ranked findings with reproduction against staging, plus fix PRs when that lane is engaged.

## 9. Self-assessment

- [ ] Permission matrix for human + machine keys  
- [ ] Automated cross-tenant tests in CI  
- [ ] Webhook SSRF lab results recorded  
- [ ] Invoice state machine diagram shared with eng  
- [ ] Hot spend path (if any) has dual control  
- [ ] Reorg/credit policy written  
- [ ] Secrets scan clean on main  
- [ ] Deploy provenance queryable for last 30 days  

## 10. Related notes

- [001 — Wallet application threat model](001-wallet-app-threat-model.md)
- [002 — Metaprotocol and indexer trust](002-metaprotocol-indexer-trust.md)

## References

- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- Bitcoin Core documentation for RPC access control and wallet permissions relevant to your deployment topology
