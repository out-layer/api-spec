# Changelog

All notable changes to the OutLayer API spec. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows [SemVer](https://semver.org/) — see [docs/versioning.md](docs/versioning.md).

## [Unreleased]

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
