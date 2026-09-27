# Gate / Gateway / Provider Model

**Status:** v0.2.0 open specification.

## Overview

QRCodePay separates market availability, route selection, and real-world payment
execution into three concepts:

- **Gate:** merchant-facing market/payment product entry.
- **Gateway:** internal selectable execution route.
- **Provider / Upstream:** actual real-world execution party or implementation.

This model is jurisdiction-aware, not country-specific. Provider execution is
where available and subject to market-specific arrangements.

## Gate

A Gate represents the merchant-facing entry point for a market, product, or
payment type. It is the boundary a merchant and PAYER can reason about before
execution.

Gate responsibilities may include:

- market/country/product availability;
- merchant permission;
- supported assets;
- quote/rate configuration;
- limit/risk precheck;
- Provider availability;
- default Gateway group;
- jurisdiction-aware compliance status.

Gate is not the same as Provider. A Gate may be available or unavailable
independently from any single Provider.

## Gateway

A Gateway represents an internal selectable execution route under a Gate.

Gateway responsibilities may include:

- route selection;
- Provider health checks;
- failover / disaster recovery;
- weight / priority;
- execution adapter;
- query adapter;
- receipt normalization.

One Gate may bind to multiple Gateways. One Gateway may route to one or more
Providers.

## Provider / Upstream

A Provider or Upstream is the actual real-world execution party or implementation
used by a Gateway.

Provider responsibilities may include:

- actual real-world execution;
- execution response;
- receipt / reference;
- failure reason;
- refund or reversal capability where supported.

Provider execution is where available and subject to market-specific
arrangements. This open spec does not expose private Provider routes, partner
details, credentials, funding operations, risk rules, or reconciliation logic.

## Example

The following example is illustrative only and does not claim live regional
availability:

```
Gate: MARKET_LOCAL_EXECUTION
  Gateway group:
    - LOCAL_EXECUTION_PRIMARY
    - LOCAL_EXECUTION_BACKUP
  Provider candidates:
    - provider_default
    - provider_backup
```

An implementation might define a market-specific Gate such as
`TH_LOCAL_EXECUTION` for internal testing or configuration, but the QRCodePay open
spec is not Thailand-only and does not claim live availability in Thailand or any
other region.

## Boundaries

The open specification may describe:

- Gate identifiers;
- allowed Gate / Provider scope inside a mandate;
- normalized Provider execution status;
- receipt hashes;
- refund or reversal capability where supported.

The open specification must not include:

- production Provider endpoints;
- private routing rules;
- partner-specific details;
- Provider funding operations;
- internal risk rules;
- country-specific compliance claims.
