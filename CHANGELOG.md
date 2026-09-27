# Changelog

All notable changes to the QRCodePay Open Specification are documented here.

## v0.2.0 — Reusable wallet mandate and per-payment confirmation

- Changed Payment Mandate from order-bound authorization to reusable
  wallet-level bounded authorization.
- Added grantee, single-payment limit, cumulative limit, usage limit, validity,
  confirmation mode, authorization nonce, and optional scope fields.
- Added a separate per-payment confirmation and mandate usage model.
- Added `confirmation_hash` to Audit Record.
- Replaced superseded refund states with the canonical Payment Intent refund
  lifecycle.
- Changed stablecoin-only wording to supported digital-asset terminology.
- Added `confirmPayment` to the Provider Interface concepts.

## v0.1.1 — Smart Wallet / AI Agent positioning and Refund Claim specification

- Added smart-wallet authorization specification.
- Added Gate / Gateway / Provider model.
- Added refund claim and PAYER verification specification.
- Added refund lifecycle states to Audit Record.
- Added smart-wallet / AI-agent threats and refund-specific threats.
- Clarified QRCodePay is not ordinary crypto checkout or merchant balance system.
- Clarified `payment_intent_id` vs `payment_intent_hash`.

## v0.1.0 — Initial public specification draft

- Initial public specification draft.
- Added Payment Mandate v0.1 JSON Schema.
- Added Audit Record v0.1 JSON Schema.
- Added provider interface boundary (open boundary only).
- Added threat model.
- Added Payment Mandate and Audit Record examples.
