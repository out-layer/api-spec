# OutLayer API Spec

OpenAPI 3.1 specification for the OutLayer HTTP API.

- **Live docs**: https://api.outlayer.fastnear.com/docs (Scalar UI)
- **Raw spec**: https://api.outlayer.fastnear.com/openapi.json
- **TypeScript SDK**: [`@outlayer/sdk`](https://www.npmjs.com/package/@outlayer/sdk) — source at [out-layer/sdk-js](https://github.com/out-layer/sdk-js)

## Coverage

| Area | Status |
|------|--------|
| Wallet (custody, multi-chain) | v0.1 |
| Policy (limits, approvals, freeze) | v0.1 |
| Multisig approvals | v0.1 |
| Audit log | v0.1 |
| Execution (HTTPS / blockchain triggers) | v0.2 (planned) |
| Secrets | v0.2 (planned) |
| Vault (sovereign custody) | v0.2 (planned) |
| Scheduler | v0.2 (planned) |

## Validate locally

```bash
npm install
npm run lint       # @redocly/cli lint
npm run preview    # @scalar/cli preview
```

## License

MIT
