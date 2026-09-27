# Refund Claim / PAYER Verification

**Status:** v0.2.0 open specification.

## Overview

QRCodePay does not automatically refund contract payments to the original sending
address by default. The original sender may be an exchange wallet, shared wallet,
custody address, or address that cannot safely receive refunds for the PAYER.

If a contract payment has been received but Provider execution fails, the order
may enter a refundable state. The merchant/integrator may initiate a refund claim
by submitting payer email information. QRCodePay sends verification to the PAYER.
The PAYER verifies the claim and submits their own refund wallet address. Refund
execution should occur only to the PAYER-confirmed address.

Merchant-supplied payer email is an initiation signal, not proof of payer
identity and not authorization to choose the final refund wallet address.

## Why refund claims exist

Real-world payment execution may fail after contract payment has been received.
Stopping at a Provider failure state is not enough. If funds were received, the
order should eventually resolve to:

- payment completed;
- refund completed; or
- a manual terminal state.

Refund Claim gives QRCodePay a PAYER-verified path to return funds without
assuming the original sending address is safe.

## Merchant vs PAYER roles

- **Merchant / Integrator:** may initiate the refund claim by supplying order
  reference and payer email information.
- **PAYER:** verifies the claim and supplies the refund wallet address.
- **QRCodePay:** records refund status and links the claim to the Audit Record.
- **Provider:** may provide execution failure evidence or reversal capability
  where supported.

The merchant must not control the final refund wallet address.

## Refund claim lifecycle

Canonical refund-related Payment Intent states:

- `payment_failed`
- `refund_claim_requested`
- `refund_email_sent`
- `refund_review_pending`
- `refund_processing`
- `refunded`
- `refund_failed`
- `manual_terminal`

`refund_review_pending` begins only after the verified PAYER submits the refund
wallet, chain, asset, and required verification data. It is the first manual
review point.

## Required fields

A Refund Claim should include:

- `refund_claim_id`
- `payment_intent_hash`
- `order_hash`
- merchant reference
- payer email hash or verification channel reference
- refund asset
- `chain_id`
- refund amount
- status
- timestamps

`chain_id` SHOULD use the same CAIP-2 form as the Payment Mandate, for example
`eip155:8453`.

Plaintext payer email should not be placed in public Audit Records.

## PAYER verification

PAYER verification confirms that the party receiving the refund flow controls the
verification channel. Verification links or OTPs should expire, and refund claim
IDs should be single-use.

Verification does not remove the need for chain, asset, amount, and address
validation before refund execution.

## Refund wallet address submission

The verified PAYER submits the refund wallet address. The merchant may not choose
or override it.

The implementation should validate:

- chain;
- asset;
- amount;
- address format;
- duplicate refund execution;
- high-risk claim flags.

## Audit fields

Audit linkage should include:

- `payment_intent_hash`
- `order_hash`
- `merchant_hash` where applicable
- `mandate_hash`
- `transfer_tx_hash`
- `provider_receipt_hash` where applicable
- `refund_claim_id`
- `refund_tx_hash` where applicable
- status
- status timestamp

The Audit Record should link refund claim state to the same payment proof path.

## Security considerations

- Merchant-supplied email is only an initiation signal.
- PAYER verification is required.
- Refund wallet address is submitted only by the verified PAYER.
- Refund links / OTPs expire.
- Refund claim IDs are single-use.
- Refund execution must be idempotent.
- `refund_tx_hash` must be unique.
- The order must have confirmed contract payment before refund is available.
- Manual review should be available for high-risk claims.
- Chain, asset, and amount must be validated before refund execution.
- Audit Record links refund claim to `payment_intent_hash` and `order_hash`.

## Out of scope

- Closed refund processor implementation
- Real Provider routing
- Production email provider
- Licensing or regulatory strategy
- Custody or merchant balance design
- Guaranteed refund availability in every region
