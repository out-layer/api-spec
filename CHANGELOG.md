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
- `DepositIntentRequest.BySourceAsset` branch — request body now formally
  accepts `{ source_asset, destination_asset?, amount }` in addition to the
  legacy `{ chain, token, amount }` shape; both shapes are modelled as
  `anyOf` branches.
- `DepositIntentResponse.deposit_address` description now enumerates the
  per-chain address format (NEAR 64-char hex, EVM `0x`+40 hex, Solana
  base58, Bitcoin `bc1…`/`1…`/`3…`) so clients can validate the address
  format client-side before initiating a transfer.

### Changed

- `TransferRequest.to` is now the canonical recipient field. The legacy
  field name `receiver_id` is retained as a deprecated alias — existing
  clients sending `receiver_id` continue to work, new clients should use
  `to` to match `WithdrawRequest.to` and the dashboard. Sending both
  fields in the same body is rejected with a 400 (`duplicate field`
  deserialization error). This closes the API inconsistency where
  `/wallet/v1/transfer` required `receiver_id` while
  `/wallet/v1/intents/withdraw` required `to` — a foot-gun discovered
  during e2e sweep where a client using `to` for both got a confusing
  `missing field receiver_id` 400 on transfer.
- `RequestStatusResponse.result` is now `anyOf [WithdrawResult, null, object]`
  with a description tying the shape to `type`. Non-breaking — existing
  clients that treated `result` as opaque continue to work; clients consuming
  `result.delivered` for `type = "withdraw"` now have a typed reference to
  validate against.
- `DepositIntentRequest` is now an `anyOf` of two object shapes
  (`BySourceAsset`, `ByChainAndToken`) rather than a single object with
  `[chain, amount, token]` required. The legacy `[chain, amount, token]`
  combination still validates against the `ByChainAndToken` branch
  (`token` is now optional with a default of `"USDC"`, matching the
  coordinator behavior — clients sending `token` continue to work).
- `DepositIntentResponse.expires_at` and `.estimated_time_secs` are no
  longer in `required`. The coordinator omits these fields when 1Click does
  not return them, which the previous schema marked as a spec violation
  (clients that strictly validated `required` rejected valid responses).
- `DefuseAssetId` and `DestinationAsset` schemas added and used by
  `DepositIntentRequest` so the `destination_asset` default isn't
  duplicated across the two `anyOf` branches.

### Behavior

- **`createDepositIntent` now returns a chain-appropriate `deposit_address`
  for every source chain.** Previously the coordinator silently ignored
  the (undocumented but accepted) `source_asset` request field and
  defaulted `chain` to `"solana"`, so every cross-chain origin returned a
  Solana base58 address — a **lose-funds risk** for EVM/Bitcoin/NEAR
  callers who would send tokens to an address on the wrong chain. The
  legacy `{ chain, token }` shape also now accepts `chain="near"` (was
  rejected with HTTP 400). Schema unchanged for the legacy shape; only
  observable response values changed. Fixes
  [fastnear/near-outlayer#25 Issue A](https://github.com/fastnear/near-outlayer/issues/25).

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
