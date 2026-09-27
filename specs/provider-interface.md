# Provider Interface (Open Boundary)

**Status:** v0.2.0 open interface concepts.

This document defines the **open interface boundary** between QRCodePay's
authorization layer and real-world payment execution. It describes the shape of
method concepts a Provider integration must expose so that smart-wallet or
AI-agent authorization can become auditable Provider execution.

## What this is — and is not

> **This is an interface boundary only.**
>
> This is **not** the closed QRCodePay.io provider implementation. It deliberately
> contains **no** production API endpoints, payment routing internals, provider
> API keys, partner details, provider funding operations, risk rules, or
> reconciliation logic. Those remain proprietary and out of scope.

The open layer specifies request/response concepts and the hashes that flow
across the boundary. It does not imply live Provider availability, does not define
ordinary crypto checkout, and does not define merchant balances or reusable user
balances.

## Conventions

- All hashes are lowercase `0x`-prefixed keccak-256 unless noted.
- Timestamps are RFC 3339 / ISO 8601 UTC.
- Amounts are decimal strings.
- Method signatures below are language-neutral; transport
  (HTTP/JSON-RPC/MCP/internal SDK) is not mandated by this version.
- Errors should use a stable `{ code, message }` shape; specific codes are left
  to a later revision.

## Smart Wallet / AI Agent execution boundary

Merchant API signing authenticates the merchant request. It does **not**
authorize payment.

Payment authorization comes from the PAYER, a smart wallet / smart account
policy, and the contract-bound payment flow:

1. Merchant or integrator requests a Payment Intent.
2. The final PAYER wallet is identified when the customer enters the payment
   flow and proves wallet control.
3. The wallet matches a reusable Payment Mandate or authorizes a new one.
4. The PAYER or wallet policy confirms the concrete Payment Intent, creating one
   mandate usage.
5. Contract payment emits or anchors `payment_intent_hash`, `mandate_hash`, and
   confirmation evidence.
6. Provider execution and receipts are linked into an Audit Record.

The v0.2 profile requires confirmation for every concrete payment. Confirmation
is separate from reusable mandate authorization and may be supplied by the PAYER
or an allowed smart-wallet policy.

## Gate / Gateway / Provider model

The interface separates market entry, route selection, and actual execution:

- **Gate:** merchant-facing market/payment product entry. It represents
  availability, merchant permission, supported assets, quote/rate configuration,
  limits, Provider availability, and jurisdiction-aware compliance status.
- **Gateway:** internal selectable execution route. It handles route selection,
  Provider health, failover, priority, execution adapters, query adapters, and
  receipt normalization.
- **Provider / Upstream:** actual real-world execution party or implementation.
  It returns execution responses, receipts, failure reasons, and refund/reversal
  capability where supported.

Gate is not the same as Provider. One Gate may bind to multiple Gateways. One
Gateway may route to one or more Providers. Provider execution is where available
and subject to market-specific arrangements.

See [`gate-gateway-provider.md`](gate-gateway-provider.md).

## Method concepts

These method concepts are intentionally not bound to a production endpoint or
transport. Implementations may expose them through an internal service, SDK, MCP
tool surface, or other integration layer.

### `createPaymentIntent`

Create a structured Payment Intent from a merchant/integrator request.

- **Input:** merchant request reference, order reference, amount, asset, Gate
  preference, optional metadata.
- **Output:** internal `payment_intent_id` if used by the implementation,
  `payment_intent_hash`, and the canonical intent payload the hash commits to.
- **Notes:** The intent is a request. It carries no payment authorization on its
  own.

### `getPaymentIntent`

Return a canonical Payment Intent view.

- **Input:** `payment_intent_hash` or internal `payment_intent_id`.
- **Output:** canonical intent fields, status, quote expiry where applicable, and
  associated Gate / Provider scope.
- **Notes:** Public audit references should use `payment_intent_hash`.

### `getQuote`

Return an indicative quote for executing a given intent.

- **Input:** `payment_intent_hash` or canonical intent.
- **Output:** quoted asset/amount, expected local rail class, quote expiry, and
  Gate availability indicator.
- **Notes:** Indicative only. This interface does not define pricing, FX, or
  routing internals.

### `requestMandate`

Produce the reusable wallet-level Payment Mandate to be signed or validated by
the PAYER wallet / smart account.

- **Input:** PAYER wallet, grantee, chain, asset, `confirmation_mode`,
  `single_limit`, `total_limit`, `usage_limit`, validity window, authorization
  nonce, optional Gate / Provider scope, and revocation setting.
- **Output:** unsigned Payment Mandate object and EIP-712 typed-data payload where
  applicable.
- **Notes:** The PAYER authorizes the canonical mandate, not the agent's raw
  prompt and not the merchant API signature. A reusable mandate does not
  inherently commit to one Payment Intent or order.

### `confirmPayment`

Confirm one concrete Payment Intent under an active reusable mandate.

- **Input:** `payment_intent_hash`, `mandate_hash`, amount, chain, asset, grantee,
  confirmation nonce, and PAYER or wallet-policy proof.
- **Output:** confirmation status, `confirmation_hash`, and mandate usage
  reference.
- **Notes:** Confirmation must be idempotent. One Payment Intent may consume at
  most one accepted mandate usage.

### `executeProviderPayment`

Request Provider execution after authorization and contract payment proof are
available.

- **Input:** `payment_intent_hash`, `mandate_hash`, `confirmation_hash`,
  `transfer_tx_hash`, Gate, Provider scope, and normalized execution request.
- **Output:** execution status, Provider reference, optional
  `provider_receipt_hash`, failure reason if applicable.
- **Notes:** This method concept does not expose closed Provider routing or risk
  rules.

### `queryProviderExecution`

Query the latest Provider execution state.

- **Input:** `payment_intent_hash`, `mandate_hash`, or Provider reference.
- **Output:** normalized status, receipt/reference hashes, failure reason, and
  timestamp.
- **Notes:** Query adapters normalize Provider-specific responses without
  exposing partner-specific internals.

### `submitTransferProof`

Submit evidence of the supported digital-asset transfer or contract payment.

- **Input:** `payment_intent_hash`, `mandate_hash`, `transfer_tx_hash`.
- **Output:** acceptance status referencing the mandate and Audit Record.
- **Notes:** Binds contract payment proof to the reusable mandate and the
  separately confirmed Payment Intent.

### `submitProviderReceipt`

Submit the Provider receipt hash for executed real-world payment.

- **Input:** `payment_intent_hash`, `mandate_hash`, `provider_receipt_hash`.
- **Output:** acceptance status and resulting Audit Record linkage.
- **Notes:** Plaintext receipt content stays off-chain and is shared by selective
  disclosure where needed.

### `getPaymentStatus`

Return the current lifecycle status for a payment.

- **Input:** `payment_intent_hash`, `mandate_hash`, or internal
  `payment_intent_id`.
- **Output:** Audit Record view including `status`, `status_timestamp`, contract
  payment proof, Provider receipt hash, and refund status where applicable.
- **Notes:** Status values follow the Audit Record `status` enum.

## Optional Refund Claim methods

Refund Claim methods are optional interface concepts for orders where contract
payment was received but Provider execution did not complete.

### `createRefundClaim`

Create a refund claim initiation.

- **Input:** `payment_intent_hash`, `order_hash`, merchant reference, payer email
  or payer email hash, refund asset, chain, and refund amount.
- **Output:** `refund_claim_id`, status such as `refund_claim_requested` or
  `refund_email_sent`.
- **Notes:** Merchant-supplied payer email is an initiation signal only. It is not
  proof of payer identity and does not authorize the merchant to choose the final
  refund wallet address.

### `getRefundClaim`

Return refund claim state.

- **Input:** `refund_claim_id` or `payment_intent_hash`.
- **Output:** refund status, timestamps, and Audit Record linkage.

### `sendPayerVerification`

Send PAYER verification for a refund claim.

- **Input:** `refund_claim_id` and payer verification channel reference.
- **Output:** delivery status.
- **Notes:** This spec does not define a production email provider or message
  template.

### `submitRefundAddress`

Submit the PAYER-controlled refund wallet address after verification.

- **Input:** `refund_claim_id`, verified PAYER authorization, refund chain, refund
  asset, and refund wallet address.
- **Output:** accepted/rejected status.
- **Notes:** The refund wallet address should be submitted by the verified PAYER,
  not controlled by the merchant.

### `submitRefundExecutionProof`

Submit proof that refund execution completed.

- **Input:** `refund_claim_id`, `refund_tx_hash`, amount, asset, chain, and
  timestamp.
- **Output:** Audit Record status update such as `refunded`.
- **Notes:** Refund execution must be idempotent and `refund_tx_hash` must be
  unique.

### `getRefundStatus`

Return the current refund lifecycle status.

- **Input:** `refund_claim_id` or `payment_intent_hash`.
- **Output:** refund status, verification state, execution proof hash where
  present, and terminal/manual status.

## Out of scope

The following are intentionally **not** specified here:

- QRCodePay.io closed Provider implementation
- Production API endpoints
- Provider/partner API credentials and routes
- Internal routing and rail selection rules
- Provider funding operations
- Risk controls and fraud rules
- Reconciliation internals
- Merchant balances or reusable user balances
- Ordinary crypto checkout
- Country-specific compliance claims
