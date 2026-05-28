# Changelog

All notable changes to the OutLayer API spec. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows [SemVer](https://semver.org/) — see [docs/versioning.md](docs/versioning.md).

## [Unreleased]

### Added

- `WithdrawResult` schema — typed result payload for a successful `withdraw`
  request. Documents `intent_hash` and `delivered` fields previously emitted
  unrecorded by the coordinator. Returned in `result` of
  `GET /wallet/v1/requests/{id}` and in the `result` of the
  `request_completed` webhook for withdraw requests.
- `delivered` documented as `"native_near"` or `"nep141:<contract>"` — see
  issue
  [fastnear/near-outlayer#25](https://github.com/fastnear/near-outlayer/issues/25)
  for the bug this resolves (the coordinator used to emit `"wnear"` for every
  NEP-141 transfer, including USDC, regardless of the actual on-chain effect).

### Changed

- `RequestStatusResponse.result` is now `anyOf [WithdrawResult, null, object]`
  with a description tying the shape to `type`. Non-breaking — existing
  clients that treated `result` as opaque continue to work; clients consuming
  `result.delivered` for `type = "withdraw"` now have a typed reference to
  validate against.

### Behavior

- **Multisig-approved withdraws now run the same pre-checks as the
  synchronous path** (recipient storage / balance / account-existence).
  Approved withdraws to a non-existent named NEAR account now fail with
  `status = "failed"` instead of silently burning the source funds. Visible
  to integrators consuming `request_completed` webhooks or polling
  `GET /wallet/v1/requests/{id}`.
- **Approval-path multisig withdraws are now explicitly NEAR-only.** A
  pending approval whose `request_data.chain` is anything other than
  `"near"` (a row that should not exist in a correct deployment, but could
  arise from old DB migrations) now resolves to `status = "failed"` with a
  clear error. Cross-chain multisig withdrawals were never wired up, so
  this surfaces a pre-existing limitation rather than introducing one.

## [0.1.0-alpha.1] — 2026-05-20

Initial public draft. Covers the wallet API surface.

### Added

- **Registration**: `POST /register` (anonymous + bound-to-account modes, vault scope reserved for v0.2).
- **Wallet read**: `GET /wallet/v1/address`, `/balance`, `/tokens`.
- **Wallet write**: `POST /wallet/v1/call`, `/transfer`, `/intents/deposit`, `/intents/withdraw`, `/intents/withdraw/dry-run`, `/intents/swap`, `/intents/swap/quote`, `/sign-message`.
- **Request tracking**: `GET /wallet/v1/requests`, `/wallet/v1/requests/{id}`.
- **Policy**: `GET /wallet/v1/policy`, `POST /wallet/v1/encrypt-policy`, `/sign-policy`, `/invalidate-cache`.
- **Approvals**: `GET /wallet/v1/pending_approvals`, `POST /wallet/v1/approve/{id}`, `/reject/{id}` (NEP-413 signed).
- **Audit**: `GET /wallet/v1/audit`.
- Shared error schema with 18 typed error codes.
- `Idempotency-Key` header parameter for all write operations.
- Auth scheme: `BearerAuth` (`Authorization: Bearer wk_...`).

### Known gaps

- Execution, Secrets, Vault, and Scheduler endpoints are intentionally deferred to v0.2/v0.3 — see [docs/coverage.md](docs/coverage.md).
- Native cross-chain transfer (`POST /wallet/v1/deposit`) is in CUSTODY.md but absent from the current coordinator router; will be added in v0.2 once the coordinator catches up.
