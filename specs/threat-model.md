# Threat Model (v0.2.0)

This document lists threats the open authorization, interface, and audit layer is
designed to address, and the mitigation each relies on. It covers the **open**
layer only. Closed Provider operations, production routing, provider funding
operations, reconciliation, and partner rails are out of scope.

These are design intentions for a v0.2.0 specification and **do not** claim a
deployed, audited, or production-integrated system.

## Smart Wallet / AI Agent threats

### Agent exceeds authorized scope

- **Risk:** A compromised or misbehaving AI Agent attempts to pay more than the
  PAYER intended or execute outside an approved route.
- **Mitigation:** A reusable bounded Payment Mandate constrains grantee, chain,
  asset, single-payment amount, cumulative amount, usage count, validity window,
  optional Gate / Provider scope, and authorization nonce. Smart-wallet policy
  validation should reject requests outside the approved scope.

### Mandate replay

- **Risk:** A mandate authorization or per-payment confirmation is replayed to
  trigger duplicate or out-of-scope execution.
- **Mitigation:** Authorization nonce and validity protect mandate creation.
  Per-payment confirmation nonce, unique mandate usage, quota checks, and
  idempotent handlers prevent duplicate execution while allowing intended reuse.

### Stale quote execution

- **Risk:** An agent or intermediary executes against an expired quote or outdated
  route assumption.
- **Mitigation:** Quote expiry and mandate validity are checked before contract
  payment and Provider execution. High-risk stale quotes should move to manual
  review or require renewed PAYER confirmation.

### Wrong Gate / Provider route

- **Risk:** Execution is routed through a Gate or Provider outside the PAYER's
  authorization or outside merchant permissions.
- **Mitigation:** Gate / Provider allowlists are part of mandate scope and
  smart-wallet policy. Provider execution must match the authorized Gate and
  Provider boundary recorded for the Payment Intent.

### Smart wallet policy bypass

- **Risk:** An implementation accepts a signature or session action without
  enforcing smart-wallet policy.
- **Mitigation:** Use ERC-1271 signature verification where applicable and require
  smart-wallet policy validation for spending limit, allowed Gate, deadline,
  nonce, chain, asset, and authorization scope.

### Compromised session key

- **Risk:** A delegated key or session key is compromised and attempts unintended
  payments.
- **Mitigation:** Session keys should inherit bounded scope, expiry, spending
  limit, allowed Gate / Provider scope, and revocation. High-risk actions should
  require human confirmation or policy confirmation.

### Misleading Payment Intent

- **Risk:** A malicious or compromised merchant creates a misleading Payment
  Intent, causing the PAYER or AI Agent to approve the wrong order.
- **Mitigation:** Merchant API signing authenticates request origin only; it is
  not payment authorization. The reusable mandate limits wallet-level scope, and
  the PAYER or wallet policy separately confirms the concrete Payment Intent.

### Prompt or tool injection

- **Risk:** AI Agent prompt/tool injection causes an unintended payment request.
- **Mitigation:** The agent can only request payment. It cannot authorize outside
  the mandate. Human confirmation or smart-wallet policy confirmation should be
  required for high-risk actions.

### Wrong chain or asset

- **Risk:** Payment proof is submitted for the wrong chain, asset, or token
  contract.
- **Mitigation:** Mandate and smart-wallet policy bind `chain_id`, `asset`, and
  amount limits. Per-payment confirmation binds the concrete Payment Intent.
  Contract events should reference `payment_intent_hash`, `mandate_hash`, and
  confirmation evidence, and handlers should validate chain and asset before
  accepting proof.

### Provider execution mismatch

- **Risk:** Provider execution differs from the authorized Payment Intent or the
  recorded contract payment.
- **Mitigation:** Provider execution is linked to `payment_intent_hash`,
  `mandate_hash`, `transfer_tx_hash`, and `provider_receipt_hash`. Mismatches
  should move to failed/manual review states.

### Audit Record inconsistency

- **Risk:** Contract payment proof, Provider receipt, and status records disagree.
- **Mitigation:** Event handling should be idempotent. Audit proof linkage binds
  `payment_intent_hash`, `order_hash`, `merchant_hash`, `mandate_hash`,
  `transfer_tx_hash`, `provider_receipt_hash`, status, and timestamp.

## Refund Claim threats

### Merchant submits attacker-controlled payer email

- **Risk:** A merchant initiates a refund claim with an email controlled by an
  attacker.
- **Mitigation:** Merchant-supplied email is only an initiation signal. It is not
  proof of PAYER identity and cannot select the final refund wallet address.

### Compromised merchant account initiates fraudulent refund claim

- **Risk:** A compromised merchant account attempts to redirect refund flow.
- **Mitigation:** PAYER verification is required before refund address submission.
  High-risk claims should require manual review and merchant request
  authentication.

### Payer email compromise

- **Risk:** An attacker controlling the PAYER's email completes refund
  verification.
- **Mitigation:** Refund claim links or OTPs expire, claim IDs are single-use, and
  high-risk claims should require additional checks or manual review.

### Refund verification phishing

- **Risk:** A user is tricked into submitting a refund wallet address through a
  fake verification flow.
- **Mitigation:** Verification UX should clearly identify the order context and
  use bounded, expiring verification links. This open spec does not define the
  production message provider or template.

### Refund claim replay

- **Risk:** A valid refund claim is reused to request another refund.
- **Mitigation:** Refund claim IDs are single-use. Refund execution must be
  idempotent and terminal states must reject duplicate execution.

### Duplicate refund execution

- **Risk:** The same refund is sent more than once.
- **Mitigation:** `refund_tx_hash` must be unique when present, refund execution
  handlers must be idempotent, and final refund states must be terminal.

### Wrong-chain refund wallet address

- **Risk:** The verified PAYER submits an address on the wrong chain or for the
  wrong asset, causing loss.
- **Mitigation:** Validate `chain_id`, asset, amount, and refund address format
  before refund execution. High-risk claims should move to manual review.

### Refund to exchange or custody original sender address

- **Risk:** Automatically refunding to the original sending address sends funds to
  an exchange, shared wallet, or custody address that cannot safely credit the
  PAYER.
- **Mitigation:** QRCodePay does not automatically refund contract payments to
  the original sending address by default. The verified PAYER submits the refund
  wallet address.

### Refund claim without confirmed contract payment

- **Risk:** A refund is requested for an order where no contract payment was
  received.
- **Mitigation:** Refund availability requires confirmed contract payment proof.
  Audit Records link refund claims to `payment_intent_hash` and `order_hash`.

### Refund execution proof spoofing

- **Risk:** A party submits fake refund execution proof.
- **Mitigation:** Refund execution proof must validate chain, asset, amount, and
  transaction uniqueness before status moves to `refunded`.

### Provider failure used to trigger unauthorized refund

- **Risk:** A `payment_failed` state is abused to trigger a refund without
  authorization.
- **Mitigation:** Provider failure is not sufficient by itself. The order must
  have confirmed contract payment, refund availability, PAYER verification, and
  validated refund address before refund execution.

## General open-layer threats

### Private key exposure

- **Risk:** An AI Agent that holds the PAYER's private key can drain funds or sign
  arbitrary transactions.
- **Mitigation:** The AI Agent never receives the private key. It receives only a
  bounded authorization result or operates through smart-wallet policy.

### Business data leakage

- **Risk:** Sensitive order, merchant, or pricing data becomes public on-chain.
- **Mitigation:** Only hashes and status references are anchored. Plaintext order,
  merchant, and receipt data remain off-chain and can be selectively disclosed.

### Merchant order enumeration

- **Risk:** Observers correlate public records to map a merchant's orders,
  customers, or volume.
- **Mitigation:** `merchant_hash` should be salted and scoped per order/mandate usage so
  equal merchants do not produce equal public hashes across payments.

## Known limitations (v0.2.0)

- Provider solvency and custody of funds in flight are operational concerns in the
  closed layer and are not addressed by this specification.
- Availability / denial-of-service of any single component is out of scope here.
- On-chain anchoring guarantees depend on contracts and chains that this
  specification does not claim are deployed.
- This specification does not claim live Safe production integration, live
  AI-agent platform integration, AP2 compatibility, x402 compatibility, or any
  official protocol partnership.
