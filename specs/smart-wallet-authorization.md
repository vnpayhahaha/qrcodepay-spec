# Smart Wallet Authorization

**Status:** v0.2.0 open specification.

## Overview

QRCodePay is designed for smart-wallet and AI-agent authorization flows, not
ordinary crypto checkout. The core authorization object is a reusable
wallet-level bounded Payment Mandate approved by the PAYER directly or validated
by a smart-wallet policy. Each concrete payment requires a separate confirmation
and mandate usage.

The open spec separates merchant request authentication from payment
authorization:

- merchant API signing authenticates a merchant or integrator request;
- PAYER authorization grants reusable bounded wallet-level permission;
- PAYER or wallet policy confirms each concrete payment;
- smart-wallet policy validates scope;
- contract payment confirms fund movement;
- Audit Record links authorization, contract payment proof, Provider receipt, and
  final status.

## Why smart-wallet authorization exists

AI Agents can create payment intent, but they should not hold a PAYER's private
key or receive unlimited spending authority. Smart wallets and account
abstraction can enforce spending limits, allowed Gate / Provider scope, deadlines,
nonces, revocation, and human or policy confirmation.

This makes QRCodePay authorization and payment execution infrastructure for
AI Agents, not a generic wallet payment gateway.

## Roles

- **Merchant / Integrator:** creates or requests a Payment Intent and authenticates
  its request. This does not authorize payment.
- **PAYER:** the user or account authority that approves payment permission.
- **AI Agent:** creates or executes payment requests within an approved mandate or
  policy.
- **Smart Wallet / Smart Account:** validates policy, signature, limits, and
  execution scope.
- **QRCodePay:** records authorization, contract payment proof, Provider receipt,
  and status linkage.
- **Provider:** performs real-world payment execution where available through the
  Provider Interface.

## Authorization layers

### 1. Merchant API signing

Merchant API signing authenticates request origin and prevents unauthorized
merchant-side creation or mutation of Payment Intent requests. It does **not**
authorize payment and must not be presented as payment permission.

### 2. PAYER Payment Mandate

The Payment Mandate authorizes a reusable bounded wallet-level scope. It should be
EIP-712-compatible where applicable so signers and verifiers can agree on typed,
canonical fields.

The mandate may constrain:

- PAYER wallet
- grantee type and identifier
- `chain_id`
- `asset`
- single-payment limit
- cumulative limit
- usage limit
- confirmation mode
- Gate / Provider scope
- validity window
- authorization nonce
- revocation capability

The reusable mandate does not inherently commit to one `payment_intent_hash`,
`order_hash`, or `merchant_hash`.

### 3. Per-payment confirmation and mandate usage

After the final PAYER wallet is known and a valid mandate is selected, the PAYER
or wallet policy confirms the concrete Payment Intent. That confirmation creates
one mandate usage and binds the Payment Intent to `mandate_hash`. The current
v0.2 profile defines `confirm_each`.

### 4. Smart Wallet / Smart Account validation

Smart wallets validate policy before contract payment or Provider execution. A
policy may check spending limit, allowed Gate, allowed Provider, chain, asset,
deadline, nonce, authorization scope, and session-key permissions.

ERC-1271 may be used for smart contract wallet signature validation where
applicable.

### 5. Contract payment

Contract payment confirms fund movement and should emit or anchor an event that
references `payment_intent_hash` and `mandate_hash`.

`payment_intent_id` may be an internal system identifier. The open chain and
audit proof path should use `payment_intent_hash`, the derived hash commitment of
the canonical Payment Intent.

### 6. Audit Record

The Audit Record links authorization, contract payment proof, Provider receipt,
refund status where applicable, and final status. It should record
`payment_intent_hash`, `order_hash`, `merchant_hash`, `mandate_hash`,
`confirmation_hash`, `transfer_tx_hash`, `provider_receipt_hash`, status, and
timestamp.

## Payment Mandate

The Payment Mandate is the reusable trust boundary between Agent requests and
wallet-level permission. The AI Agent can request payment, but the mandate
defines what the Agent may request and each concrete payment still requires its
configured confirmation.

`mandate_hash` is derived from the canonical mandate after serialization/signing
and is not a field inside the mandate object.

## AI Agent execution

An AI Agent may:

- create or request a Payment Intent;
- prepare a wallet-level mandate request when no matching active mandate exists;
- ask the PAYER or smart wallet for mandate authorization;
- request confirmation for one concrete Payment Intent;
- execute allowed payment steps inside the mandate;
- submit or query proof references.

An AI Agent must not:

- hold the PAYER's private key;
- execute outside mandate or policy scope;
- choose a Provider route outside the approved Gate / Provider scope;
- treat merchant API signing as payment authorization.

## ERC-1271 / ERC-4337 alignment

Initial design targets:

- **EIP-712:** typed Payment Mandate signing where applicable.
- **ERC-1271:** smart wallet signature validation.
- **ERC-4337 / account abstraction:** future-facing smart account execution model.

This specification does not claim live production compatibility with any specific
wallet provider and does not claim any official partnership.

## Internal ID vs hash commitment

`payment_intent_id` may exist as an internal identifier for implementation
lookups. It is not the chain/audit commitment.

`payment_intent_hash` is the derived hash commitment used in contract events,
Audit Records, and cross-system proof linkage.

## Concrete payment confirmation

The current v0.2 profile requires confirmation for each concrete payment. The
confirmation may be a PAYER action or an allowed smart-wallet policy decision;
it is separate from reusable mandate authorization. QRCodePay does not define
ordinary crypto checkout as its main path.

## Security considerations

- Bounded authorization
- Authorization nonce and validity window
- Single-payment, cumulative, and usage limits
- Per-payment confirmation nonce and unique mandate usage
- Allowed Gate / Provider scope
- Revocation
- Replay protection
- Human confirmation or policy confirmation where required
- ERC-1271 verification where applicable
- Audit Record linkage for traceability

## Out of scope

- Ordinary crypto checkout
- Custodial wallet
- Merchant balance account
- Reusable stored value
- Specific wallet provider implementation
- Live Safe production integration
- Live AI-agent platform integration
