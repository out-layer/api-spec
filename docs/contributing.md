# Contributing

How to add or modify endpoints in the OutLayer API spec.

## Prereqs

- Node 18+
- `git`
- Understanding of OpenAPI 3.1 + JSON Schema 2020-12 (most YAML in `openapi.yaml` is self-explanatory; ask in PR if unsure)

## Setup

```bash
git clone https://github.com/out-layer/api-spec
cd api-spec
npm install
```

## Add an endpoint — checklist

1. **Confirm the endpoint exists in the coordinator.** The spec describes reality, not aspiration. If you're proposing a *new* endpoint, that's a separate conversation — open an issue on the coordinator repo first.

2. **Add the path entry to `openapi.yaml`** under `paths:`. Pick a tag (Wallet, Policy, Approvals, Audit, Requests, Meta) and write a one-line `summary` + multi-line `description` (Markdown ok).

3. **Add an `operationId`** in camelCase. This becomes the method name in generated SDKs. Pick something idiomatic (`intentsWithdraw`, not `WALLET_V1_INTENTS_WITHDRAW_POST`).

4. **Reference shared response components** for errors:
   ```yaml
   '400': { $ref: '#/components/responses/BadRequest' }
   '401': { $ref: '#/components/responses/Unauthorized' }
   ```
   Don't redefine error responses inline — use the shared `components/responses/*` aliases.

5. **Define new schemas in `components/schemas/`**, not inline. Inline schemas don't generate clean SDK types. Reference them with `$ref`.

6. **Add at least one `example`** for request bodies and success responses. Examples become code samples in Scalar UI.

7. **Run validation:**
   ```bash
   npm run lint       # must report "Your API description is valid" with 0 warnings
   npm run types      # must produce a clean dist/types.ts
   ```

8. **Update [docs/coverage.md](coverage.md)** if the endpoint is in a new area.

9. **Update [CHANGELOG.md](../CHANGELOG.md)** under `[Unreleased]`.

10. **Open a PR** with:
    - The endpoint(s) added
    - The motivating use case (1–2 sentences)
    - Link to the coordinator PR or issue that implemented it

## Modify an existing endpoint

The bar is higher: existing clients depend on the current shape.

| Change | Backward-compatible? | Version bump |
|---|---|---|
| Add optional request field | ✅ | minor |
| Add response field | ✅ | minor |
| Tighten description / add example | ✅ | patch |
| Add response `enum` value | ✅ (consumers tolerate unknown) | minor |
| Remove field | ❌ | major (post-1.0: deprecation cycle first) |
| Rename field | ❌ | major |
| Change field type | ❌ | major |
| Move auth from header to body | ❌ | major |

For breaking changes pre-1.0, batch them in a `next` branch and ship together as `0.X+1.0`.

## Style

- **YAML**: 2-space indent, no tabs. `redocly format openapi.yaml` not required but consistent.
- **`snake_case` on the wire**, `camelCase` for `operationId`. The wire format matches the Rust handlers; SDKs convert per-language convention if they want.
- **Descriptions** are GitHub-flavored Markdown. Cross-link with `[link text](path)`.
- **One example per request body**, more for endpoints with mode variations (e.g., `register` has anonymous and bound-to-account modes).

## When in doubt

- **Use existing endpoints as templates.** `POST /wallet/v1/intents/withdraw` is a good reference for a write op with idempotency + multisig.
- **Keep schemas flat.** Deeply-nested response types are hard to consume in non-TS languages.
- **Ask in the PR.** Spec design is part of the API contract — bikeshedding here saves real pain later.
