# QRCodePay Open Specification

> Smart-wallet authorization and last-mile payment execution infrastructure for AI Agents.

QRCodePay is **smart-wallet authorization and last-mile payment execution
infrastructure for AI Agents**, not traditional crypto checkout.

```
Merchant requests a Payment Intent.
The final PAYER wallet matches or authorizes a reusable bounded mandate.
PAYER or wallet policy confirms the concrete payment.
Contract payment records proof.
Gate / Gateway / Provider execution resolves the real-world payment.
Audit Record preserves traceability.
```

This repository contains **only the open authorization, interface, and audit
proof layer** of QRCodePay. The QRCodePay.io provider execution layer (local
payment execution, routing, provider funding operations, risk controls, and
reconciliation) remains **proprietary and pluggable** and is intentionally
**not** part of this repository.

- **Website:** https://qrcodepay.ai/
- **Whitepaper:** https://qrcodepay.ai/whitepaper.html
- **Status:** `v0.2.0` open specification
- **Contributing:** [`CONTRIBUTING.md`](CONTRIBUTING.md)
- **Security:** [`SECURITY.md`](SECURITY.md)
- **License:** [`LICENSE`](LICENSE)

> Warning: This repository does not contain QRCodePay.io provider execution code.

---

## Why this exists

AI agents and smart wallets can express payment intent, enforce policy, and
coordinate user authorization, but real-world payment execution still needs a
careful boundary between request, authorization, fund movement, provider
execution, and audit evidence.

QRCodePay defines that boundary for order-based payments:

1. A merchant or integrator signs a request to create or request a Payment
   Intent.
2. The final PAYER wallet is identified when the customer enters the payment
   flow and proves wallet control.
3. The wallet matches an active reusable Payment Mandate or authorizes a new one.
4. The PAYER or wallet policy confirms the concrete Payment Intent, creating one
   mandate usage.
5. A contract-bound payment confirms fund movement and records
   `payment_intent_hash`.
6. QRCodePay records an Audit Record and routes Provider execution through a
   Gate / Gateway / Provider model.
7. The order reaches a completed, refunded, or manual terminal state.

Merchant API signing authenticates the merchant request. It does **not**
authorize payment. Payment permission comes from the PAYER authorization layer,
the smart-wallet policy, and the contract-bound payment flow.

## Core concepts

| Concept | Role |
| --- | --- |
| **Smart Wallet Authorization** | Policy-aware wallet validation for bounded payment permission, including limits, deadlines, allowed Gate / Provider scope, and replay protection. |
| **AI Agent Payment Execution** | Agent-driven payment requests that can execute only inside a PAYER-approved mandate or smart-wallet policy. |
| **Payment Intent** | A structured description of the requested order payment. `payment_intent_id` may exist internally; `payment_intent_hash` is the open commitment used in contract events and Audit Records. |
| **Payment Mandate** | Reusable wallet-level bounded authorization for a grantee, chain, asset, confirmation mode, amount limits, usage limit, validity window, and revocation state. It is not inherently bound to one order. |
| **Mandate Usage** | One separately confirmed Payment Intent executed under a reusable Payment Mandate. |
| **Contract-bound Payment Intent** | The payment proof path in which contract events reference `payment_intent_hash` and `mandate_hash`. |
| **Audit Record** | A hash-based record linking authorization, contract payment proof, Provider receipt, refund state where applicable, and final status. |
| **Gate / Gateway / Provider Model** | The jurisdiction-aware abstraction separating merchant-facing market products, selectable execution routes, and actual execution Providers. |
| **Refund Claim / PAYER Verification** | A refund flow in which the merchant may initiate a claim, but the verified PAYER controls the refund wallet address. |
| **Order-based execution without merchant balances** | Each payment is resolved per order. QRCodePay v1 does not define merchant balances or reusable user balances. |

The system links **Payment Intent → PAYER wallet binding → reusable Payment
Mandate → per-payment confirmation and mandate usage → contract payment →
Provider execution → Audit Record** so a verifier can connect real-world
execution to specific authorization without exposing private business data
on-chain.

## Specification documents

- [Payment Mandate schema](specs/payment-mandate.schema.json)
- [Payment Mandate example](examples/payment-mandate.example.json)
- [Audit Record schema](specs/audit-record.schema.json)
- [Audit Record example](examples/audit-record.example.json)
- [Provider Interface](specs/provider-interface.md)
- [Smart Wallet Authorization](specs/smart-wallet-authorization.md)
- [Gate / Gateway / Provider Model](specs/gate-gateway-provider.md)
- [Refund Claim / PAYER Verification](specs/refund-claim.md)
- [Refund Claim example](examples/refund-claim.example.json)
- [Threat Model](specs/threat-model.md)

## Not a crypto checkout

QRCodePay does not define an ordinary wallet checkout flow as its main path.
The v0.2 profile requires confirmation for each concrete payment. Confirmation
may come from a PAYER action or an allowed smart-wallet policy decision and is
separate from reusable mandate authorization.

QRCodePay is not:

- traditional crypto checkout
- ordinary wallet payment gateway
- USDT checkout
- merchant balance system
- reusable user balance system
- generic QR payment gateway

## No merchant balance in v1

QRCodePay v1 does not maintain merchant balances or reusable user balances. Each
payment follows an order-based route:

```
Payment Intent
→ PAYER wallet binding
→ Reusable Payment Mandate
→ Per-payment confirmation and mandate usage
→ Contract Payment
→ Provider Execution
→ Audit Record
→ completed, refunded, or manual terminal state
```

If Provider execution fails after a contract payment has been received, the order
should not stop at `payment_failed`. It should eventually resolve to a payment
completed state, a refund completed state, or a manual terminal state.

## Payment Mandate

The Payment Mandate is the core authorization object. It is intended to be signed
as **EIP-712** typed structured data where applicable so the PAYER sees readable
fields and verifiers get a deterministic typed digest. **ERC-1271** is an initial
design target so smart-contract wallets can validate signatures through their own
policy logic.

- Schema: [`specs/payment-mandate.schema.json`](specs/payment-mandate.schema.json)
- Example: [`examples/payment-mandate.example.json`](examples/payment-mandate.example.json)

`mandate_hash` is **not** a field inside the mandate. It is derived from the
canonical reusable wallet-level mandate after serialization/signing. The mandate
does not inherently commit to `payment_intent_hash`, `order_hash`, or
`merchant_hash`. Each confirmed payment creates one mandate usage, and the Audit
Record links that concrete Payment Intent to `mandate_hash` and
`confirmation_hash`.

## Audit Record

The Audit Record stores only **hashes and status references** by default. It can
be anchored on-chain as one record per payment in early versions; later versions
may batch records using Merkle roots to reduce cost while preserving proof paths.

- Schema: [`specs/audit-record.schema.json`](specs/audit-record.schema.json)
- Example: [`examples/audit-record.example.json`](examples/audit-record.example.json)

The Audit Record may also represent refund lifecycle states when contract payment
was received but Provider execution did not complete.

## Interoperability roadmap

QRCodePay is not trying to replace MCP, AP2, x402, ERC-4337, EIP-712, ERC-1271,
or wallet standards. QRCodePay is designed to connect smart-wallet and AI-agent
authorization flows with real-world payment execution through supported
Providers where available.

- **MCP Payment Server:** expose payment tools to AI agents.
- **AP2-style mandates:** designed to align with agentic payment authorization
  and mandate concepts.
- **x402-style payments:** explore compatibility for machine-to-machine
  payments, API payments, and agentic payments.
- **ERC-4337 / EIP-712 / ERC-1271:** wallet-native authorization and smart-wallet
  validation.
- **Gate / Gateway / Provider:** separate market entry, route selection, and
  actual Provider execution.

This roadmap does not claim live AP2 compatibility, live x402 compatibility, live
wallet-provider compatibility, or any official protocol partnership.

## Security notes

See [`specs/threat-model.md`](specs/threat-model.md) for the full threat model.
Summary of the load-bearing properties:

- The AI Agent never receives the PAYER's private key.
- The Payment Mandate or smart-wallet policy bounds grantee, chain, asset,
  single-payment amount, cumulative amount, usage count, validity, optional
  Gate / Provider scope, and authorization nonce.
- Each concrete payment requires a separate confirmation and unique mandate usage.
- Merchant API signing authenticates request origin only; it is not payment
  authorization.
- Contract events should reference `payment_intent_hash` and `mandate_hash`.
- Audit Records link authorization, contract payment proof, Provider receipt, and
  final state.
- Refund Claims require PAYER verification before a refund wallet address is
  accepted.

This specification describes intended designs and **does not claim** that any
contract is deployed, audited, or integrated with a specific wallet or AI-agent
platform.

## Status and roadmap

**Current status:** `v0.2.0` public specification.

Roadmap:

1. Stabilize reusable Payment Mandate typed-data and per-payment confirmation
   canonicalization rules.
2. Define contract event shapes for `payment_intent_hash`, `mandate_hash`, and
   `confirmation_hash`.
3. Expand Provider Interface conformance examples.
4. Add Refund Claim test vectors.
5. Add Audit Record anchoring examples.

See [`CHANGELOG.md`](CHANGELOG.md) for version history.

## License

Licensed under the Apache License, Version 2.0. See [`LICENSE`](LICENSE).
