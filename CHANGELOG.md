# Changelog

All notable changes to the OutLayer API spec. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows [SemVer](https://semver.org/) — see [docs/versioning.md](docs/versioning.md).

## [Unreleased]

### Upgrading a client

1. **Stop storing the trial key.** `POST /trial-key` now issues a key derived
   from the wallet's master, and `GET /wallet/v1/payment-key` answers the same
   string again under the same credential. A key issued before this release
   was random: the GET answers `409 payment_key_not_recoverable` for it, and
   the client keeps the copy it has or creates a payment key.
2. **A `Bearer near:` wallet claims the trial too.** `POST /trial-key` takes
   any wallet credential; before, only `wk_`.

### Added

- `POST /wallet/v1/sponsorship {code}` (`redeemSponsorCode`) — redeem a sponsor
  code: the code's allowance lands on the wallet's nonce-0 key for the code's
  term, the key is created if absent, the trial converted. A key carries one
  sponsor while its grant is live; after it ends another code is taken. Only the
  credential that reads the nonce-0 key redeems onto it, and the answer always
  carries `payment_key`. Refusals: `404 sponsor_code_invalid` (one answer for
  every reason), `403 payment_key_other_credential`,
  `409 sponsor_cannot_top_up`, `409 payment_key_deleted`,
  `409 payment_key_revoked`, `409 payment_key_not_recoverable`; none takes a use
  of the code.
- `GET /wallet/v1/payment-key` (`getPaymentKey`) — the nonce-0 key, derived
  again: `{payment_key, owner, nonce, expires_at, subscription}`;
  `404 no_payment_key`, `403 payment_key_other_credential`,
  `409 payment_key_revoked`, `409 payment_key_not_recoverable`.
- `TrialKeyResponse`, `PaymentKeyResponse`, `SponsorshipResponse` as named
  schemas; `TrialKeyRefusal.reason` gains `no_payment_key`,
  `payment_key_not_recoverable`, `sponsor_code_invalid`,
  `sponsor_cannot_top_up`, `payment_key_deleted`.

### Changed

- The nonce-0 key is bound to the credential that claimed it: only that
  credential reads it (`403 payment_key_other_credential`), and once that
  `wk_` is revoked the key stops working on `/call` and the GET answers
  `409 payment_key_revoked`.
- `/call` to `hyperliquid` or `polymarket` from a wallet with an owner uses
  the owner's policy row: attached when the body names none, refused for any
  other account (`403 policy_row_not_owner`); repeating that answers
  `403 calls_suspended` on those connectors for a while. With no policy in the
  run, they trade on their built-in default.
- `POST /trial-key`: `409 trial_already_claimed` points to
  `GET /wallet/v1/payment-key`. A refused credential now answers in the wallet
  API's shape (`ErrorResponse`), as on every `/wallet/v1/*` route, instead of
  `TrialKeyRefusal` `unauthorized`.
- A granted subscription — an operator's gift or a sponsor code — lifts the
  free-tier custody ceiling (`operation_limit_reached` on `custody:*`) as a
  bought one does. The bare trial is still under it.
- The one-allowance-call-at-a-time rule (`429 call_already_in_flight`) counts
  a sponsored key's in-flight calls against its code's `max_parallel`.

## [0.1.0-alpha.3] — 2026-10-03

### Upgrading a client

What an integration has to change for the entries below. Each point is safe
against the current server and the previous one. Mainnet before this release
ran the server of 2026-09-28, which predates 0.1.0-alpha.2 as well: an
integration coming from it reads both sections, and the points below are the
ones that change how a money call is made and followed.

1. **Send `X-Idempotency-Key`**, not `Idempotency-Key`, on every write — one
   key per operation, minted and persisted on your side before sending. The
   server now reads both names. Before, it ignored the second, and a retried
   write ran again.
2. **Send `X-Answer-Within: <seconds>` below your client's timeout** on
   `intentsWithdraw`, `intentsTransfer`, `intentsSwap`, `createLimitOrder`,
   `createPaymentCheck`, `batchCreatePaymentChecks`, `claimPaymentCheck` and
   `reclaimPaymentCheck`. The call then answers `status: processing` with
   `request_id` and `poll_url` within that many seconds of the request's
   arrival (`createPaymentCheck`: `creating` with the check's `poll_url`)
   instead of outliving your timeout. `0` to `80`; absent, the call waits its
   own budget as before (up to 80 s). The wait only shortens: the operation
   runs to its outcome whatever you waited. Out of range → `400 bad_request`
   before anything runs.
3. **Read `status`, not the HTTP code.** A `200` can answer `processing`: the
   money was handed over and its outcome is not known yet. Do not retry it; that
   is how a payment is made twice. Poll `poll_url` (`GET
   /wallet/v1/requests/{request_id}`) until the status is terminal. This applies
   to `intentsWithdraw`, `intentsSwap`, `intentsTransfer`,
   `claimPaymentCheck`, `reclaimPaymentCheck`, `createLimitOrder` (the
   `LimitOrderProcessing` answer) and every confidential op. It also applies to
   anything that credits a user, ships goods or releases funds on a call's
   success: do that on the terminal status.
4. **Terminal request statuses:**
   - `success`, or `completed` for a payment check leg;
   - `failed`. With `result.never_executed` or `result.never_submitted` set,
     nothing moved, and a retry with a NEW idempotency key is safe. Without
     them, funds may have moved: a bridge that failed or refunded
     (`result.reason`), or a NEAR transaction that failed on chain after its
     gas was spent. Reconcile the balance before acting again;
   - `refunded` (confidential);
   - `needs_review`: the outcome could not be established. Do not retry; see
     the request's `result.reason`.
5. **Payment checks:**
   - **Create.** Keep `check_key` whatever `status` says. `creating` means the
     funding is unconfirmed: the check cannot be claimed or reclaimed until it
     reads `unclaimed` (poll `poll_url`). `failed` means it was never funded.
   - **Claim and reclaim.** A strict decoder must make `remaining`,
     `claimed_at` and `reclaimed_at` optional: they are absent while the answer
     is `processing`. New fields: `request_id`, `status`, `poll_url`.
   - **Batch create.** Can answer `200` with fewer checks than asked and an
     `error`. The checks listed exist and are funded or being funded; the rest
     were not created.
   - **Status and list.** New check statuses: `creating`, `claiming`,
     `reclaiming`, `failed`.
6. **A key is held by the request that reserved it** — including one that
   ended `failed` after the reserve (a short balance, a recipient that does not
   exist). A refusal before the reserve — authentication, a malformed body, the
   policy, `wallet_busy`, a bad `X-Answer-Within` — holds nothing, and the same
   key may be sent again. Retry a `failed` with `never_executed` or
   `never_submitted` under a NEW key; any other `failed`, or `needs_review`: do
   not — it may have moved funds (item 4); reconcile the balance first.
7. **Recover after a timeout or a dropped connection by re-sending with the
   same key.** The answer is HTTP `200` with `error: duplicate_idempotency_key`
   and the request it belongs to: `request_id`, `type`, `status`, `created_at`,
   and `result`, `updated_at`, `poll_url` (while not terminal) as the request
   has them; `message` reads as before. For a payment-check create or batch,
   `checks: [{check_id, check_key, status}]` — the checks that key made, each
   with its key derived again (`null` when asked with another API key of the
   wallet than the one that created it). Branch on `error` (the code is 200),
   read `request_id` and `status`, poll `poll_url` while present. The re-send
   may instead meet `409 wallet_busy` with `in_flight_request_id` (`null` while
   the request is being written: retry in a moment), or the plain answer. Act —
   credit, ship, release funds — on the terminal status only.
8. **Webhooks.** A `request_completed` webhook can arrive after the call that
   started the request has answered.

### Fixed

- `Approver.pubkey` is optional, as the server always treated it. With it, the
  approver votes only with that key; without it, with any access key of the
  account, or through its wallet contract.

- `approveRequest`, `rejectRequest`: `public_key` must be a full-access key of
  `account_id` on chain; otherwise `401 invalid_signature`, and nothing is
  stored. Before, any key could vote in an approver's name and take that
  approver's vote, and a function-call key held by an application could vote
  as the approver. A chain that cannot be read answers `503` with `Retry-After`.
- `Bearer near:`, `register` with a NEAR proof and `PUT /wallet/v1/api-key`:
  the signing key must be a full-access key of the account (or the key of an
  implicit account not created yet). A function-call key is refused.
- `approveRequest`, `rejectRequest`: a second vote by the same account, in
  either direction, is `409 already_approved`. A reject after an approve
  answered `500`.
- `approveRequest`, `rejectRequest`: a vote on an approval that is no longer
  pending is `409 conflict` naming its state. It answered `500`.
- `rejectRequest`: a reject from an approver without a pinned key now cancels
  the request at once, as one from a pinned approver did. The keystore already
  honoured it as a veto; the request sat in `pending_approval` until it expired.
- Approval counts (`approved_count`, `approval_count`, and the threshold) count
  approve votes only. A reject vote from anyone used to count toward the
  threshold, so one approve plus a stranger's reject could start an execution
  the keystore then refused.
- `approveRequest`, `rejectRequest`: a vote from an account the wallet's policy
  does not admit is `403 not_approver` and is not stored — an account not
  listed, one pinned to another key, or a pinned one voting through a wallet
  contract. A stranger's approve used to count toward the threshold, and a
  stranger's reject was stored and answered `reject_vote_recorded`. A refusal is
  decided on the policy as the chain holds it, so an approver added a moment
  ago is admitted; a chain that cannot be read is `503`.

- **The idempotency header is `X-Idempotency-Key`.** The `IdempotencyKey`
  parameter named it `Idempotency-Key`; the coordinator now reads both, the
  documented name first. A write retried under the old name used to run again
  instead of answering `duplicate_idempotency_key`.
- **No error after funds may have moved.** A synchronous withdraw, swap or
  transfer that fails after handing its funds over answers
  `status=processing` with a `poll_url`, not an HTTP 5xx a caller would retry.
- **Every late settlement is announced.** A request settled after its call
  answered sends its `request_completed` webhook once, whether a status read
  or the background settler closed it.

### Changed

- The `X-Idempotency-Key` description states what holds a key: the request
  that reserved it, including one that ended `failed` after the reserve; a
  refusal before the reserve holds nothing.
- **Payment checks, limit orders and confidential ops settle past their
  response.** Every transfer they hand over is recorded before it leaves and
  settled from the chain (checks, limit order funding) or the confidential
  status (confidential ops) when its call stops watching.
  - `createPaymentCheck` / `batchCreatePaymentChecks`: a check carries `status`
    — `unclaimed`, or `creating` + `poll_url` while its funding is unconfirmed —
    and its `check_key` is returned either way. A batch is checked against the
    balance whole before any check is funded, and one that stops part way lists
    the checks it created with `error` naming the rest.
  - `claimPaymentCheck` / `reclaimPaymentCheck` answer `request_id` and
    `status`; `processing` + `poll_url` while the transfer is unconfirmed
    (`remaining` and the timestamp then absent). Both take `X-Idempotency-Key`.
    New check statuses: `creating`, `failed`.
  - `createLimitOrder` can answer `LimitOrderProcessing` (`order_id`,
    `poll_url`); an order is cancelled for an unexecuted funding only once the
    chain shows it never executed.
  - Confidential ops can answer `processing` + `poll_url`.
- **Withdraw, swap and transfer settle past their response.** A synchronous
  `intentsWithdraw` (same-chain included), `intentsSwap` or `intentsTransfer`
  whose settlement outlasts its wait answers `status=processing` with a
  `poll_url` (new on `SwapResponse`) and settles on its own. The operation runs to its outcome
  even if the caller disconnects. In the request's `result`,
  `settled_late: true` marks one settled from the chain afterwards, and a
  `failed` one with `never_executed: true` or `never_submitted: true` moved
  no funds.

### Added

- **`X-Answer-Within: <seconds>`** on the money operations that wait
  (`intentsWithdraw`, `intentsTransfer`, `intentsSwap`, `createLimitOrder`,
  `createPaymentCheck`, `batchCreatePaymentChecks`, `claimPaymentCheck`,
  `reclaimPaymentCheck`): how long the call waits before answering
  `processing` with `request_id` and `poll_url` (`createPaymentCheck`:
  `creating`). Whole seconds, `0` to `80`, counted from the request's arrival;
  absent, the full wait as before. Read on every wallet route; acts on these.
  Out of range → `400 bad_request` before anything runs.
- **The `duplicate_idempotency_key` answer names its request.** Beside `error`
  and `message` (unchanged), the body carries `request_id`, `type`, `status`,
  `created_at`, and `result`, `updated_at` and `poll_url` (while not terminal)
  as the request has them; for a payment-check create or batch, `checks` with
  each check's `check_id`, `check_key` (`null` when the asking API key is not
  the one that created it) and status. A batch's request is `processing` while
  it runs and lists the checks reserved so far, `completed` once it answered.
  If the keys cannot be derived again (the keystore does not answer) the
  re-send is refused like any keystore call and can be sent again.
- **Votes from contract wallets.** `approveRequest` and `rejectRequest` accept a
  second body, `ContractVoteAuth` `{account_id, authorization}`, from an
  approver without access keys: a NEP-616 wallet contract owned by an EVM key
  or a passkey. The wallet's own `w_resolve_auth` must resolve `authorization`
  to the vote message. Accepted from wallet builds the deployment lists; a vote
  from any other account is `400`, a vote the wallet does not resolve to this
  message is `401 invalid_signature` naming the wallet's reason.
- `ApprovalDetail.approvers[].proof_kind`: `nep413` or `contract`.
  `approvers[].signature` is `null` for a contract vote.

## [0.1.0-alpha.2] — 2026-10-02

### Added

- **Notices.** `TaskKind` gains `notice`: a task that tells the owner
  something and asks nothing. `InboxTask.reply_pubkey` is `null` for one.
  `POST /inbox/tasks/{id}/acknowledge` (Got it) closes an open notice as
  `done`, inside the session, with no signature and no event; approve and
  reject of a notice are 400 `invalid_request`, an approval before its nonce
  is spent. The event of a new notice is `task_created` with `kind: notice`.

- **`GET /public/connectors/{id}/describe` carries `callers`**: the manifest's
  block of which doors a run may come through (`direct`, `contract` — `allow`,
  `deny` or `{only: [...]}` — `https`, `meta_tx`), as written; `null` when the
  manifest declares none.
- **Inbox: tasks between an agent and its owner.** `POST`/`DELETE /inbox/session`
  (an owner's session opened by one NEP-413 statement signed by a full-access
  key — for an implicit account not on chain yet, its own key; a custody
  wallet signs with `/wallet/v1/sign-message`; five devices an account), `GET /inbox/tasks` (ciphertext for the
  device, `more` when the list is cut), `/inbox/tasks/{id}` (delete),
  `/reject`, `/files/{n}`, `/origin`, `/inbox/mutes`, `/inbox/devices`,
  `/inbox/webhook` (a secret told once, events signed with it). Withdrawing
  another device and naming or removing the webhook take an `OwnerConfirmation`:
  one signature over a sentence naming the action and the minute. Refusals
  `InboxRefusal` with `reason` among `session_required`, `session_replaced`,
  `invalid_statement`, `task_not_found`, `task_closed`, `invalid_request`,
  `confirmation_required`, `upstream_unavailable`, `internal_error`.
- **`GET /inbox/tasks/{id}/origin` carries `input`**: what the run was asked,
  byte for byte as the attestation's `input_hash` is the SHA-256 of it —
  the call's `input` as the coordinator serialised it, keys in the caller's
  order. A string for a call over HTTPS; `null` for a request on chain, whose
  transaction carries it.
- **`GET /wallet/v1/pending_approvals_by_pubkey` is told to the wallet's owner,
  signed in** (`session_required`, `session_replaced`, `not_wallet_owner`). One
  approval by its id (`GET /wallet/v1/approval/{id}`) needs no session, as
  before.

### Fixed

- **A confidential quote without an estimate is an answer, not an error.** A
  confidential `swap-quote` or `withdraw/dry-run` whose 1Click quote carries no
  `amountOut` answers `200` without `amount_out` / `min_amount_out` and with a
  `hint` saying 1Click gave no estimate for the route. Asking again quotes the
  same way.
- **Polling an unknown call is a 404.** `GET /calls/{call_id}` answered `500
  internal_error` for an id no call has. It now answers `404` with `reason`
  `call_not_found`.
- **A database failure answers a fixed sentence, never the database's text.**
  `/call`, `/calls/{call_id}`, `/payment-keys/balance`, `/subscription/*` and
  `/wallet/v1/*` put Postgres's own message — table, constraint, value — into
  `error` / `message` when a query failed. They now answer `503` + `Retry-After`
  when the database could not serve the request right now (pool exhausted,
  connection lost, server shutting down, serialization failure, deadlock), as
  `upstream_unavailable`, and `500 internal_error` otherwise, each with a fixed
  sentence. `/public/*` reads answer the same split with their own 503
  sentences.
- **`Idempotency-Key` described as it behaves.** The parameter said a repeated
  key "returns the original result". It never did: the server answers `200`
  with `duplicate_idempotency_key` and the original `request_id` in `message` —
  a pointer, read through `GET /wallet/v1/requests/{request_id}`. Documentation
  only; the behaviour is unchanged since the first wallet release.
- **A token a chain does not carry is the caller's error, not ours.**
  `/wallet/v1/deposit-intent`, its confidential twin, and the cross-chain
  `withdraw` pre-flight answered `500` ("1Click API may be unavailable") when
  `(chain, token)` named a token that chain has no 1Click asset for — the
  commonest way to hit it being the `USDC` default on a chain without USDC.
  They now answer `400 unsupported_token` naming what that chain does carry
  (the code was already in `ErrorCode`; nothing emitted it), and reserve `503`
  + `Retry-After` for the case where the token catalog itself could not be
  read. `POST /wallet/v1/withdraw/dry-run` keeps reporting it as
  `would_succeed: false, reason: bridge_rejected` rather than a 4xx.
- **Agent Connect** — `POST /wallet/v1/binding/events` checks the shared secret
  before reading the body. An unauthenticated caller is answered `401`
  whatever it sends, where a malformed body used to be answered `422` — the
  code this API reserves for `OnChainTxFailed`. An authenticated caller whose
  body cannot be read gets `400`.

### Added

- **`upstream_unavailable` on `/wallet/v1/*`** — added to `ErrorCode`: `503` +
  `Retry-After`, this deployment's database could not serve the request right
  now. The name `/call` already answers the condition with.

- **Limit orders** — `POST/GET /wallet/v1/limit-orders`,
  `GET /wallet/v1/limit-orders/{order_id}`, `POST …/{order_id}/cancel`,
  `POST /wallet/v1/limit-orders/cancel-all`. A swap rested on 1Click at the
  owner's price, funded from the wallet's intents balance. The wallet
  authorises it once and the payout happens later with no further signature,
  so it is gated as the exit it can become: a default-DENY `limit_order`
  capability and transaction type of its own, the address rules on `recipient`, and
  the per-token amount limit — `swap` and `cross_chain_withdraw` neither imply
  it nor stand in for it. The endpoints are a thin door onto 1Click's
  `/v0/orders`: the request carries its parameters (`recipient`,
  `recipient_type`, …) and the answer is its order — field names in snake_case,
  enumerated values in lower case, nothing renamed; `is_payout_status_final` is
  the only terminal signal. Multisig wallets answer `pending_approval` and
  approvers sign over the order's terms. Freezing a wallet stops new orders but
  does not cancel resting ones; cancelling is never frozen (`cancel-all`), and
  is asynchronous, so a last slice may still fill.
  `limit_order` joins `RequestType` and `in_flight_operation`.
- **Robinhood Chain (`hood`)** on `Chain` and `WithdrawChain`: an Arbitrum L2,
  EVM like the rest — it shares the wallet's one derived `0x` address — and
  bridged both ways through 1Click, whose identifier for it is `hood`. It does
  not carry USDC, the `token` default of `/wallet/v1/deposit-intent`, so a
  deposit intent by `(chain, token)` must name one of its tokens.
- **Agent Connect** — `status_reason` on `BindingResponse`: why a binding is
  not `active`, as the fault class of the last observation. Absent while
  active.
- **Agent Connect** — `registry_disagrees` on `AgentConnectDeniedResponse.class`:
  a leased account whose collection's `nft_token` does not confirm the
  account's own `nft_item_info`. Reversible (`terminal: false`); the registry
  may trail the account by a block.
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

- **Cross-chain deposit refunds follow the wallet policy.** The refund address
  is the policy's `refund_addresses.<chain>` entry; without one, the wallet's
  derived address on the chain where it has one (NEAR, Solana, EVM chains,
  HyperCore — the request's `refund_address` is ignored and `hint` says so),
  else the request's `refund_address`. Without a policy, the request's
  `refund_address`, else the derived address. A chain the wallet has no
  address on, with neither, answers `400`. Same rule on the confidential
  deposit's HyperCore refunds.
- **A synchronous `/call` whose job is already queued never answers `503`.** A
  database failure while it waits answers `500 internal_error` without
  `Retry-After`: the job may still run and be charged, and a resend would run a
  second call. `503 upstream_unavailable` stays for failures before the job is
  queued.
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
  [out-layer/outlayer#25](https://github.com/out-layer/outlayer/issues/25)
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
  [out-layer/outlayer#25 Issue A](https://github.com/out-layer/outlayer/issues/25).
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
