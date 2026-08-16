# Changelog

All notable changes to the OutLayer API spec. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows [SemVer](https://semver.org/) — see [docs/versioning.md](docs/versioning.md).

## [Unreleased]

### Added

- **Execution** (`POST /call/{owner}/{project}`, `GET /calls/{call_id}`) — the
  route every project is reached through, connectors included. Documents the
  universal `operation` field a connector must name inside `input` (the one
  value the contract prices, the coordinator bills, the worker refuses to run
  without and the guest dispatches on), the `X-Use-Owner-Secret` switch, and
  the choice between waiting and polling: `"async": true` returns at once with
  `call_id` + `poll_url` and has **no window at all**, which is where long work
  belongs.
- **Subscriptions and payment keys** — `POST /wallet/v1/create-payment-key`
  (including the keyless AGENT key, one per wallet, with no plaintext in the
  response), `GET /wallet/v1/subscription/purchase-info`, and
  `GET /subscription/status` with the allowance, the balance and the connectors
  in scope.
- **A secret left for an agent** — `GET /wallet/v1/agent-secret/pubkey`,
  `POST /wallet/v1/agent-secret` (the agent's wallet pays) and
  `POST /wallet/v1/agent-secret/prepare` (a call for a named payer to send, the
  signature bound to them).
- **`PaymentKeyAuth`** security scheme (`X-Payment-Key: {owner}:{nonce}:{key}`),
  which most of these routes accept alongside `Bearer wk_`.

### Changed

- **A synchronous call that outruns its window now answers `408`**, carrying
  `call_id`, `reason: "timeout"` and a `poll_url`, instead of `500`. The outcome
  is settled, not a fault of ours — and a `500` invites the one wrong move,
  since the original may still be running and be charged, so a retry buys a
  second charge for one piece of work.
- **`POST /wallet/v1/create-payment-key`** distinguishes its failures: `402`
  when the wallet is short (naming what it holds, what is needed and where to
  send it), `400` for an amount no account could hold, and `503` when the
  balance could not be READ — which is not the same as a balance of zero, and
  used to be reported as one.

- **`ConfidentialOpResponse`** — documented the settled `result.swap_details`
  block on the request row (`intentHashes`, `nearTxHashes`,
  `originChainTxHashes`, `destinationChainTxHashes` — camelCase inner keys,
  all arrays of **plain hash strings** — plus settled amounts and refund
  fields). Upstream
  1Click switched the `*ChainTxHashes` elements from plain strings to
  `{hash, explorerUrl}` objects (2026-07-11); the coordinator normalizes them
  back to plain strings, so the OutLayer wire shape is unchanged and stable
  regardless of upstream changes. For an external-chain `confidentialWithdraw`,
  `destination_chain_tx_hashes` carries the actual destination-chain delivery
  transaction. Description-only change — no schema or route changes.

### Added

- **`destination_tx_hash` in gasless results** — cross-chain withdraw
  (`intents_cross_chain_withdraw`) and gasless swap request rows now carry a
  nullable `result.destination_tx_hash`: the real delivery transaction on the
  destination chain (sourced from 1Click's `destinationChainTxHashes`).
  `null` until the bridge settles; filled by the lazy on-read refresh and
  included in the `request_completed` webhook payload. Documented on
  `RequestStatusResponse.result`. Additive — no version bump, no breaking
  change.
- **Confidential intents** (`Confidential` tag) — 9 new routes mirroring
  `/wallet/v1/intents/*` against the Defuse confidential private shard
  (the `intents.far` contract, distinct from public `intents.near`):
  `POST /wallet/v1/confidential/{deposit,unshield,withdraw,withdraw/dry-run,transfer,swap,swap/quote,deposit-intent}`
  and `GET /wallet/v1/confidential/balance`. New schemas:
  `ConfidentialShieldRequest`, `ConfidentialUnshieldRequest`,
  `ConfidentialWithdrawRequest`, `ConfidentialSwapRequest`,
  `ConfidentialDepositIntentRequest`, `ConfidentialDepositIntentResponse`,
  `ConfidentialTransferRequest`, `ConfidentialOpResponse`,
  `ConfidentialBalanceResponse`, `ConfidentialBalanceEntry`,
  `ConfidentialBalancesResponse`. New `503` response component
  `ServiceUnavailable` + `ErrorCode`s `confidential_unavailable` /
  `confidential_jwt_expired`. All confidential routes return `503` unless the
  deployment has `ENABLE_CONFIDENTIAL_INTENTS` and the confidential partner
  agreement configured. Privacy properties (balances are real on-chain state
  on a separate private shard, the `intents.far` contract, which has no public
  RPC so it cannot be read externally — not off-chain and not a solver
  database; internal transfer/swap leave no public-chain trace, while
  SHIELD/UNSHIELD and cross-chain in/out are the only edges that touch the
  public chain) are documented in the route descriptions and CUSTODY docs —
  see the agent integration guide
  [`docs/CONFIDENTIAL_INTENTS.md`](https://github.com/out-layer/coordinator/blob/main/docs/CONFIDENTIAL_INTENTS.md)
  in the coordinator repo (also linked via `externalDocs` on the `Confidential`
  tag in this spec). Each wallet
  has a single confidential identity (the custody wallet itself); there is no
  separate or unlinkable confidential identity.
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
- `DepositIntentResponse.hint` (optional string) — non-binding advisory
  when the coordinator can suggest a more direct endpoint for the same
  logical operation. Currently emitted only for NEAR-source deposits to
  point clients at `POST /wallet/v1/intents/deposit` (one-tx
  `ft_transfer_call`, no 1Click solver hop). Backward-compatible —
  existing clients that don't read the field are unaffected.
- **Off-chain EVM signing** (`Wallet` tag) — 3 new routes for signing EVM
  payloads with the custody wallet's secp256k1 key without ever building or
  broadcasting a transaction:
  `POST /wallet/v1/evm/sign-typed-data` (EIP-712 v4),
  `POST /wallet/v1/evm/sign-message` (EIP-191 `personal_sign`), and
  `POST /wallet/v1/evm/sign-transaction` (raw tx — the **client** serializes
  the unsigned transaction; the keystore only keccak256-hashes and signs it,
  performing no assembly, nonce/gas selection, or broadcast; for EIP-1559 the
  returned `yParity` is `v - 27`). New schemas:
  `EvmSignTypedDataRequest`, `EvmSignMessageRequest`,
  `EvmSignTransactionRequest`, and `EvmSignResponse` (65-byte `0x` `r‖s‖v`
  signature, `v ∈ {27, 28}`, low-s). All signatures use a hand-rolled EIP-712
  encoder (no alloy/ethers). New policy capability `EvmSignCapability` with
  `evm_sign` **default-DENY under a policy** (set `allowed:true` to permit; a
  wallet with no policy is unrestricted; `sign_message` is the only default-allow
  capability) and a `raw_tx` sub-flag **default-OFF** gating raw-tx signing;
  `requires_approval` is not supported for `evm_sign`. Note that an
  EIP-712 signature is itself fund-moving (EIP-3009 ≈ transfer, EIP-2612 ≈
  approve), so `evm_sign` grants full authority over the EVM address float —
  bounded to whatever is bridged there; the NEAR-intents balance is never
  exposed. `GET /wallet/v1/address` now serves all supported EVM chains
  (`ethereum`, `polygon`, `base`, `arbitrum`, `optimism`, `bsc`, `avalanche`,
  plus aliases `eth`/`pol`/`matic`/`arb`/`op`/`avax` via the `Chain` enum),
  returning one shared secp256k1 `0x` address; Solana stays gated and account
  delete stays NEAR-only. Broadcast, gas, and nonce remain the client's
  responsibility — the keystore and coordinator never build or broadcast an
  EVM transaction.

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
- **Coordinator no longer auto-issues `storage_deposit` on `intents.near`
  or on the OutLayer contract.**
  - `/wallet/v1/intents/deposit` previously attempted a NEP-145
    `storage_deposit` on `intents.near` before the `ft_transfer_call`.
    intents.near uses NEP-245 multi-token storage and auto-registers
    callers via its own `ft_on_transfer` hook — the NEP-145 call was
    always failing on-chain and wasting ~0.00125 NEAR per request. The
    call is now omitted entirely; the actual deposit still works because
    of the auto-registration on first transfer.
  - `createPaymentKey` previously attempted a `storage_deposit` on the
    OutLayer contract (`state.contract_id`) for `owner`. The OutLayer
    contract creates the owner's storage entry in its `store_secrets`
    call (which still runs earlier in the flow), so the extra
    `storage_deposit` was a duplicate / no-op that often failed
    on-chain.
  - **`createPaymentKey` retains the auto-`storage_deposit` on the
    stablecoin (USDC) contract** for the integrator's convenience — that
    one is a standard NEP-141 registration paid for from the integrator's
    own wallet NEAR, and is idempotent.
- **Coordinator now structured-logs every keystore-call failure** with a
  `WARN` line carrying status code + body (for non-2xx) or transport
  error detail. Operators who tail coordinator logs will see actionable
  diagnostics for every `502 keystore_error` response (previously the
  causes were silently swallowed and only the response code was visible
  via `tower_http`). Not directly observable to integrators.

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
