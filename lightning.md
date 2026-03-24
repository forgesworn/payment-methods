# Lightning Payment Method

**Method identifier:** `lightning`

**Registry:** HTTP Payment Methods (IANA)

**Parent specification:** [draft-ryan-httpauth-payment-01](https://datatracker.ietf.org/doc/draft-ryan-httpauth-payment/)

**Status:** Proposed

---

## Overview

The `lightning` payment method enables HTTP Payment Authentication using the Bitcoin Lightning Network. The server issues a BOLT11 invoice in the challenge; the client pays it and presents the payment preimage as proof of settlement. Verification is stateless — a single SHA-256 hash comparison with no network call.

Lightning payments settle in seconds, cost fractions of a penny, and require no identity or account. This makes the `lightning` method ideal for machine-to-machine API payments, AI agent commerce, and metered service access.

## Challenge Request

The challenge request is a JSON object embedded (base64url-encoded, JCS-canonicalised) in the `request` parameter of the `WWW-Authenticate: Payment` header.

```http
WWW-Authenticate: Payment id="<hmac>",
    realm="api.example.com",
    method="lightning",
    intent="charge",
    request="<base64url of charge request below>",
    expires="2026-03-24T12:15:00Z"
```

### Charge Request Schema

```json
{
  "amount": "1000",
  "currency": "sat",
  "methodDetails": {
    "invoice": "lnbc10u1pj...",
    "paymentHash": "a1b2c3d4e5f6...64 hex chars",
    "network": "mainnet"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | Yes | Amount in the smallest unit of `currency`. |
| `currency` | string | Yes | Currency code. MUST be `sat` (satoshis). |
| `methodDetails.invoice` | string | Yes | BOLT11 payment request string. |
| `methodDetails.paymentHash` | string | Yes | 64-character lowercase hex SHA-256 hash. |
| `methodDetails.network` | string | Yes | Lightning network: `mainnet`, `testnet`, `signet`, or `regtest`. |

**JSON Schema:** [`schemas/lightning-charge-request.json`](schemas/lightning-charge-request.json)

## Credential Payload

The credential payload is included in the `payload` field of the `Authorization: Payment` credential.

```json
{
  "preimage": "f1e2d3c4b5a6...64 hex chars"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `preimage` | string | Yes | 64-character lowercase hex Lightning payment preimage. |

**JSON Schema:** [`schemas/lightning-charge-payload.json`](schemas/lightning-charge-payload.json)

## Verification Procedure

Verification is **stateless** — no database lookup or network call required.

1. Parse the `Authorization: Payment` header. Decode the base64url credential.
2. Verify the HMAC-bound challenge ID (per parent draft §4.3).
3. Check the challenge `expires` field. Reject if expired.
4. Decode the `request` parameter to extract `methodDetails.paymentHash`.
5. Validate the `preimage` field: MUST be exactly 64 lowercase hex characters.
6. Compute `SHA256(preimage)` where the preimage is decoded from hex to 32 raw bytes.
7. Compare the result to `paymentHash` using a **timing-safe comparison**.
8. If equal, the credential is valid. If not, reject.

```
SHA256(hex_decode(preimage)) == hex_decode(paymentHash)
```

The preimage is a cryptographic proof that the Lightning invoice was paid. Knowledge of the preimage is equivalent to proof of payment — no further confirmation is needed.

## Settlement Procedure

Lightning payment settlement is **atomic**. Settlement occurs on the Lightning Network when the payer reveals the preimage to the payee's node during the HTLC resolution. The preimage in the credential is proof that this settlement already occurred.

No separate settlement step is required by the server. The verification step (SHA-256 comparison) is sufficient to confirm payment.

Servers that need to confirm on-chain settlement (e.g. for high-value transactions) MAY additionally call their Lightning node's `checkInvoice` API to verify the invoice status. This is OPTIONAL and NOT REQUIRED for standard verification.

## Security Considerations

### Challenge Binding

All challenges MUST use the HMAC-SHA256 binding mechanism defined in the parent draft (§4.3). This prevents challenge parameter tampering. The server's HMAC secret MUST be at least 32 bytes of cryptographically random data.

### Transport Security

Servers and clients MUST use TLS (HTTPS). The preimage is a bearer credential — anyone who intercepts it can present it as proof of payment.

### Preimage Single-Use

Each Lightning payment produces a unique preimage. The same preimage cannot be used to pay two different invoices. However, a preimage CAN be replayed against the same server for the same challenge (since verification is stateless). Servers MUST implement replay protection:

- **Per-request mode:** Track settled payment hashes. Reject credentials for already-settled hashes.
- **Credit mode:** Allow reuse within a credit balance (the macaroon-based L402 approach handles this separately).

### Network Validation

Servers MUST include the `network` field in every challenge. Clients that only support mainnet MAY omit network validation but MUST reject any challenge where `network` is not `mainnet`. Multi-network clients (e.g. testing tools) SHOULD maintain an explicit network configuration.

### Invoice Expiry

BOLT11 invoices have their own expiry (typically 1 hour). The challenge `expires` field and the invoice expiry are independent. Servers SHOULD set the challenge expiry to be shorter than or equal to the invoice expiry.

## Reference Implementations

| Implementation | Language | Notes |
|----------------|----------|-------|
| [@forgesworn/toll-booth](https://github.com/forgesworn/toll-booth) | TypeScript | IETF Payment rail (`src/core/ietf-payment.ts`). 5 Lightning backends. |
| [Aperture PR #220](https://github.com/lightninglabs/aperture/pull/220) | Go | Lightning Labs implementation (open, not yet merged). |
