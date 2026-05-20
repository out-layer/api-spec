# Versioning policy

The spec follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html). This page defines what counts as a major / minor / patch change.

## Version sources

The version is recorded in two places that must match:

1. `openapi.yaml` → `info.version`
2. `package.json` → `version`

A pre-commit hook (planned) will block desync.

## Major version bump (`X.0.0`)

Reserved for **breaking changes**. After v1.0:

- Removing an endpoint
- Renaming an endpoint or a request/response field
- Removing a required field
- Changing a field's type (`string` → `number`)
- Narrowing a response field (e.g., removing a value from an `enum`)
- Changing an authentication scheme or moving auth from header to body

Pre-1.0, breaking changes can happen on minor bumps (this is standard SemVer). Production users should pin to a patch.

## Minor version bump (`0.X.0`)

**Additive, non-breaking** changes:

- Adding a new endpoint
- Adding an optional request field
- Adding a new field to a response (existing consumers ignore it)
- Adding a new value to a response `enum` (consumers handle it as the unknown case)
- Adding a new error code

Note: adding a new value to a **request** enum is a minor change for the spec but may cause clients with stricter validation to reject the call. We document new enum values in the changelog so SDK maintainers can update.

## Patch version bump (`0.0.X`)

**No semantic change**:

- Documentation (descriptions, examples)
- Tightening descriptions without changing behavior
- Fixing typos
- Adding `operationId` where it was missing (technically affects SDK method names — when it does, bump minor)

## Pre-1.0 (current)

This is `0.1.x` — alpha. The shape is stable enough to build against, but **expect minor versions to occasionally have small breaks** (e.g., field renames as we tighten the schema). The changelog calls out every break explicitly.

Production users:

- **Pin a patch** in `package.json`: `"@outlayer/sdk": "0.1.0-alpha.1"`, not `"^0.1.0"`.
- **Read the changelog** before upgrading minor versions.

After v1.0, breaks land only on major bumps with a deprecation cycle.

## Deprecation cycle (post-1.0)

When an endpoint or field needs to be removed:

1. **Announce** — add `deprecated: true` in the spec; document the alternative in the description.
2. **6-month grace** — the deprecated thing keeps working. Clients see warnings in Scalar UI and IDE auto-complete.
3. **Remove** — at the major version bump after the grace period.

Hot fixes for security issues skip the grace period and ship as immediate major bumps.

## Branch policy

- `main` — current released version
- `next` — staged changes for the next minor version
- Hot-fixes go to `main`; everything else goes to `next` first
