# GreyBound Research

Engineering research from [Greybound](https://greybound.com) on Bitcoin systems that run on mainnet.

Greybound builds, audits, and secures Bitcoin infrastructure. This repository publishes the threat models and hardening frameworks we use on real wallets, metaprotocol products, and production backends.

These notes are technical working documents—not marketing one-pagers, not CVE dumps, and not client reports. Engagement findings stay private until a vendor has patched and disclosure is agreed.

## Notes

| Note | Topic |
|------|--------|
| [001](notes/001-wallet-app-threat-model.md) | Wallet application threat model — assets, boundaries, scenarios, controls, review method |
| [002](notes/002-metaprotocol-indexer-trust.md) | Metaprotocol & indexer trust — sign-time binding when opinion layers sit above Bitcoin |
| [003](notes/003-bitcoin-app-backend-hardening.md) | Bitcoin app backend hardening — tenants, webhooks, invoices, RPC, deploy trust |

[Research directions](directions.md)

## About

- Site: [greybound.com](https://greybound.com)
- Org: [github.com/Grey-Bound](https://github.com/Grey-Bound)

## License

[MIT](LICENSE)
