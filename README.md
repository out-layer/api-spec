# OutLayer API Spec

OpenAPI 3.1 specification for the OutLayer HTTP API.

This repo is the **single source of truth** for the OutLayer wire format. The TypeScript SDK ([`@outlayer/sdk`](https://github.com/out-layer/sdk-js)), the dashboard docs, and any future language bindings consume this spec.

- **Interactive docs**: https://api.outlayer.fastnear.com/docs (Scalar UI)
- **Raw spec**: https://api.outlayer.fastnear.com/openapi.json
- **TypeScript SDK**: [`@outlayer/sdk`](https://www.npmjs.com/package/@outlayer/sdk)
- **OutLayer documentation**: https://outlayer.fastnear.com/docs

## Coverage

| Area | Status | Notes |
|---|---|---|
| Wallet (custody, multi-chain) | ✅ v0.1 | Address, balance, transfer, withdraw, swap, sign-message |
| Policy (limits, allowlists, freeze) | ✅ v0.1 | Encrypt → sign → on-chain store flow |
| Multisig approvals (NEP-413) | ✅ v0.1 | listPending, approve, reject |
| Audit log | ✅ v0.1 | Full event history |
| Request tracking | ✅ v0.1 | Get/list async requests |
| Execution (HTTPS / blockchain triggers) | 🚧 v0.2 | `/call/{owner}/{project}` endpoints, project deploys |
| Secrets | 🚧 v0.2 | Encrypted env vars per WASM execution |
| Vault (sovereign per-customer custody) | 🚧 v0.2 | Atomic deploy, recovery state machine, MPC CKD |
| Scheduler | 🚧 v0.3 | Interval / storage-diff / webhook triggers |

See [docs/coverage.md](docs/coverage.md) for detailed endpoint counts and the rationale behind what's deferred.

## Consume the spec

### TypeScript SDK (recommended)

```bash
npm install @outlayer/sdk
```

```ts
import { OutlayerClient } from '@outlayer/sdk';
const client = new OutlayerClient({ apiKey: 'wk_...' });
await client.withdraw({ chain: 'ethereum', to: '0x...', amount: '1000000', token: 'nep141:usdt.tether-token.near' });
```

### Auto-generate types in any language

```bash
# TypeScript
npx openapi-typescript https://api.outlayer.fastnear.com/openapi.json -o types.ts

# Python
pip install openapi-python-client
openapi-python-client generate --url https://api.outlayer.fastnear.com/openapi.json

# Go
go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest
oapi-codegen -generate types https://api.outlayer.fastnear.com/openapi.json > types.go

# Rust
cargo install progenitor-impl --bin progenitor
```

The spec uses OpenAPI 3.1.0 (JSON Schema 2020-12 compatible), so any modern generator works.

### Browse the spec

- **Scalar UI** (interactive, with code samples): https://api.outlayer.fastnear.com/docs
- **Spec file**: [`openapi.yaml`](openapi.yaml)

## Local development

```bash
git clone https://github.com/out-layer/api-spec
cd api-spec
npm install

npm run lint       # @redocly/cli — schema validation
npm run preview    # opens Redoc preview in browser
npm run bundle     # produces dist/openapi.json
npm run types      # produces dist/types.ts (for testing)
```

To validate a change before sending a PR:

```bash
npm run lint       # must pass with 0 errors and 0 warnings
npm run types      # must produce a clean types.ts
```

## Versioning

Spec versions follow [SemVer](https://semver.org/). See [docs/versioning.md](docs/versioning.md) for the change policy.

- **Major** (1.0.0 → 2.0.0): removing endpoints, removing required fields, narrowing response types
- **Minor** (0.1.0 → 0.2.0): adding endpoints, adding optional request fields, broadening response types
- **Patch** (0.1.0 → 0.1.1): documentation, examples, additive enum values

Pre-1.0, expect minor versions to occasionally break — `0.x` is alpha. Pin to a patch in production.

## Contributing

See [docs/contributing.md](docs/contributing.md) for how to add or update endpoints.

The short version: edit `openapi.yaml`, run `npm run lint && npm run types` until both pass, open a PR describing **what** changed (new endpoint, new field, etc.) and **why** (linked issue, feature request, or coordinator PR).

## License

MIT.
