# Coverage

What's in the spec, what's not, and why.

## In scope (v0.1)

**22 endpoints** across wallet, policy, approvals, audit, and request tracking.

### Wallet (12 endpoints)

| Method | Path | Purpose |
|---|---|---|
| GET | `/wallet/v1/address` | Derive address per chain |
| GET | `/wallet/v1/balance` | Read balance |
| GET | `/wallet/v1/tokens` | Token catalog |
| POST | `/wallet/v1/call` | NEAR function call |
| POST | `/wallet/v1/transfer` | Native chain transfer |
| POST | `/wallet/v1/intents/deposit` | FT into intents.near |
| POST | `/wallet/v1/intents/withdraw` | Gasless cross-chain withdraw |
| POST | `/wallet/v1/intents/withdraw/dry-run` | Simulate withdraw |
| POST | `/wallet/v1/intents/swap` | 1Click cross-chain swap |
| POST | `/wallet/v1/intents/swap/quote` | Swap price preview |
| POST | `/wallet/v1/sign-message` | NEP-413 / raw signing |

### Registration (1 endpoint)

| Method | Path | Purpose |
|---|---|---|
| POST | `/register` | Create wallet, return API key |

### Policy (4 endpoints)

| Method | Path | Purpose |
|---|---|---|
| GET | `/wallet/v1/policy` | Current decrypted policy + usage |
| POST | `/wallet/v1/encrypt-policy` | TEE-encrypt a policy JSON |
| POST | `/wallet/v1/sign-policy` | TEE-sign for on-chain submission |
| POST | `/wallet/v1/invalidate-cache` | Drop no-policy cache after on-chain change |

### Approvals (3 endpoints)

| Method | Path | Purpose |
|---|---|---|
| GET | `/wallet/v1/pending_approvals` | List pending multisig |
| POST | `/wallet/v1/approve/{id}` | Submit approval (NEP-413) |
| POST | `/wallet/v1/reject/{id}` | Submit rejection (NEP-413) |

### Request tracking & audit (3 endpoints)

| Method | Path | Purpose |
|---|---|---|
| GET | `/wallet/v1/requests` | List async ops |
| GET | `/wallet/v1/requests/{id}` | Single op status |
| GET | `/wallet/v1/audit` | Full event history |

## Out of scope (v0.1) — coming later

### v0.2 candidates

- **Execution API** — `POST /call/{owner}/{project}` for triggering WASM execution from outside NEAR; payment-key auth.
- **Secrets API** — encrypted env vars per WASM execution, accessed via the TEE.
- **Vault (sovereign per-customer custody)** — `/customer/derive-tee-key`, `/customer/sign-verification`, `/customer/register`. The atomic 5-action NEAR deploy is fundamentally a multi-step flow; documenting it cleanly takes more design than just translating the HTTP endpoints.

### v0.3+ candidates

- **Scheduler** — interval / storage-diff / webhook triggers for autonomous WASM execution.
- **Cross-chain deposit** — `POST /wallet/v1/deposit` for "send me on Ethereum, settle in NEAR Intents".
- **Webhook subscription management** — register and rotate webhook endpoints.

## Internal endpoints (intentionally not exposed)

These exist in the coordinator but are not part of the public API:

- `POST /internal/wallet-check` — worker-to-coordinator policy check
- `POST /internal/wallet-audit` — worker-to-coordinator audit logging
- `POST /internal/vault-event` — on-chain event ingestion

They use a separate auth scheme (`X-Internal-Wallet-Auth`) and are subject to change without notice.

## Versioning impact

When a v0.2 area lands, it ships as a minor version bump — additive only. Removing or renaming a v0.1 endpoint would be a major version bump and trigger a deprecation cycle (announce → 6-month deprecation period → remove).
