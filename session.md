# Session Intent

**Intent identifier:** `session`

**Registry:** HTTP Payment Intents (IANA)

**Parent specification:** [draft-ryan-httpauth-payment-01](https://datatracker.ietf.org/doc/draft-ryan-httpauth-payment/)

**Status:** Proposed

---

## Overview

The `session` intent enables deposit-based prepaid access with streaming balance deduction, top-up, and close-with-refund. It is designed for use cases where per-request payment round-trips are too slow or too expensive:

- **LLM token streaming** — AI inference APIs where tokens are generated over seconds
- **Ridesharing metering** — fare accumulates in real-time during a journey
- **Real-time data feeds** — continuous API access billed by consumption
- **Metered compute** — pay-as-you-go processing with unpredictable duration

The session intent extends the parent draft's single-request charge model into a multi-request lifecycle: the client deposits funds upfront, uses a bearer token for subsequent requests, tops up if the balance runs low, and closes the session to receive a refund of unspent funds.

## Session Lifecycle

```
    ┌──────────┐
    │ Challenge │ ← Server issues deposit invoice
    └────┬─────┘
         │ Client pays invoice
         ▼
    ┌──────────┐
    │   Open   │ ← Client sends preimage + return invoice
    └────┬─────┘
         │ Server creates session
         ▼
    ┌──────────┐     ┌──────────┐
    │  Active  │────►│  Top-up  │ ← Client pays fresh invoice
    │          │◄────│          │   Server increases balance
    └────┬─────┘     └──────────┘
         │
         ├─── Client closes ──────► Close (refund unspent)
         │
         └─── TTL expires ────────► Auto-close (refund attempted)
```

## Challenge Wire Format

```http
WWW-Authenticate: Payment id="<hmac>",
    realm="api.example.com",
    method="lightning",
    intent="session",
    request="<base64url of session challenge request>",
    expires="2026-03-24T12:15:00Z"
```

### Session Challenge Request

```json
{
  "amount": "20000",
  "currency": "sat",
  "methodDetails": {
    "depositInvoice": "lnbc200u1pj...",
    "paymentHash": "a1b2c3d4e5f6...64 hex chars",
    "network": "mainnet",
    "sessionTtlSeconds": 86400,
    "requireReturnInvoice": true
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | Yes | Deposit amount in smallest unit of `currency`. |
| `currency` | string | Yes | Currency code (e.g. `sat`). |
| `methodDetails.depositInvoice` | string | Yes | BOLT11 invoice for the deposit payment. |
| `methodDetails.paymentHash` | string | Yes | 64-character lowercase hex SHA-256 hash. |
| `methodDetails.network` | string | Yes | Lightning network identifier. |
| `methodDetails.sessionTtlSeconds` | integer | Yes | Session lifetime in seconds. RECOMMENDED: 86400 (24 hours). |
| `methodDetails.requireReturnInvoice` | boolean | Yes | Whether the server requires a return invoice for refunds. |

**JSON Schema:** [`schemas/session-challenge-request.json`](schemas/session-challenge-request.json)

## Actions

The session intent defines four actions. Each action uses a different credential payload structure, all sharing the `Authorization: Payment` header.

### Open

The client pays the deposit invoice and opens a session.

```json
{
  "action": "open",
  "preimage": "f1e2d3c4...64 hex chars",
  "returnInvoice": "lnbc1pj..."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | MUST be `"open"`. |
| `preimage` | string | Yes | 64-character hex preimage of the deposit invoice. |
| `returnInvoice` | string | Conditional | Amountless BOLT11 invoice for refunds. Required when `requireReturnInvoice` is true. |

**Session ID:** The session ID is the payment hash of the deposit invoice — i.e. `SHA256(preimage)`. This is not an independent value; the server derives it from the preimage.

**Return invoice validation:** If the return invoice contains an encoded amount, the server MUST reject the session open request. Only amountless invoices are accepted, allowing the server to set the refund amount at close time.

**JSON Schema:** [`schemas/session-open-payload.json`](schemas/session-open-payload.json)

### Bearer

The client uses the session for authenticated requests. The deposit preimage serves as a bearer token.

```json
{
  "action": "bearer",
  "sessionId": "a1b2c3d4...64 hex chars",
  "preimage": "f1e2d3c4...64 hex chars"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | MUST be `"bearer"`. |
| `sessionId` | string | Yes | 64-character hex session ID (= payment hash of deposit). |
| `preimage` | string | Yes | 64-character hex deposit preimage (bearer token). |

**Verification:** On every bearer request, the server MUST verify `SHA256(preimage) == sessionId`. The server then looks up the session and atomically deducts the request cost from the balance.

**Balance check:** The deduction MUST be atomic: `deposit - spent >= cost` in a single operation (e.g. SQL `WHERE` clause). This eliminates time-of-check-to-time-of-use (TOCTOU) race conditions.

**JSON Schema:** [`schemas/session-bearer-payload.json`](schemas/session-bearer-payload.json)

### Top-up

The client adds funds to an active session.

```json
{
  "action": "topUp",
  "sessionId": "a1b2c3d4...64 hex chars",
  "topUpPreimage": "b2c3d4e5...64 hex chars"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | MUST be `"topUp"`. |
| `sessionId` | string | Yes | 64-character hex session ID. |
| `topUpPreimage` | string | Yes | 64-character hex preimage of the top-up invoice. |

The server issues a fresh invoice for the top-up amount. The client pays it and provides the new preimage. The server verifies `SHA256(topUpPreimage) == topUpPaymentHash` and atomically increases the session balance.

**JSON Schema:** [`schemas/session-topup-payload.json`](schemas/session-topup-payload.json)

### Close

The client ends the session and receives a refund of unspent funds.

```json
{
  "action": "close",
  "sessionId": "a1b2c3d4...64 hex chars",
  "preimage": "f1e2d3c4...64 hex chars"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | MUST be `"close"`. |
| `sessionId` | string | Yes | 64-character hex session ID. |
| `preimage` | string | Yes | 64-character hex deposit preimage. |

**Close procedure:**
1. The server atomically marks the session as closed (preventing further bearer requests).
2. The server computes `refund = totalDeposited - totalSpent`.
3. If `refund > 0` and a return invoice was provided, the server pays the return invoice with the refund amount.
4. The server responds with a `Payment-Receipt` containing the refund details.

**Receipt on close:**
```json
{
  "status": "success",
  "method": "lightning",
  "timestamp": "2026-03-24T12:30:00Z",
  "reference": "a1b2c3d4...session ID",
  "refundSats": 15000,
  "refundStatus": "succeeded"
}
```

`refundStatus` values: `succeeded` (refund paid), `failed` (refund payment failed), `skipped` (no return invoice or zero balance).

**JSON Schema:** [`schemas/session-close-payload.json`](schemas/session-close-payload.json)

## Error Responses

Errors use [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) Problem Details format in the response body.

| Condition | HTTP Status | Error code | Description |
|-----------|-------------|------------|-------------|
| Session balance exhausted | 402 | `session_exhausted` | Balance insufficient for request cost. Includes `balance` and `suggestedTopUp`. |
| Session expired | 402 | `session_expired` | Session TTL exceeded. Client must open a new session. |
| Invalid bearer preimage | 401 | `invalid_credentials` | `SHA256(preimage) != sessionId`. |
| Invalid top-up preimage | 402 | `invalid_topup_preimage` | Top-up preimage does not match the issued invoice. |
| Session already closed | 409 | `session_already_closed` | Cannot perform actions on a closed session. |

Example error response:
```json
{
  "type": "https://paymentauth.org/problems/session_exhausted",
  "title": "Session balance exhausted",
  "status": 402,
  "detail": "Session balance is 0 sat. Top up to continue.",
  "sessionId": "a1b2c3d4...",
  "balance": 0,
  "suggestedTopUp": 5000
}
```

## Streaming Exhaustion

When a session balance is exhausted mid-stream (e.g. during an SSE response or WebSocket connection), the server MAY emit a `NeedTopUp` event rather than abruptly terminating the stream.

```json
{
  "type": "needTopUp",
  "sessionId": "a1b2c3d4...",
  "currentBalance": 0,
  "suggestedAmount": 5000
}
```

**Delivery mechanism by transport:**

| Transport | Delivery |
|-----------|----------|
| Server-Sent Events (SSE) | SSE event with `event: needTopUp` |
| WebSocket | JSON message |
| Standard HTTP | Server SHOULD return `402` with a session-aware challenge instead |

The client can respond by sending a top-up action, then resuming the stream.

## Session TTL and Expiry

Servers MUST define a session TTL. The RECOMMENDED default is 86400 seconds (24 hours). The TTL is communicated in the challenge request (`sessionTtlSeconds`) and confirmed in the open receipt.

When a session expires:
1. The server atomically marks the session as closed.
2. If a return invoice was provided and the remaining balance is > 0, the server SHOULD attempt a refund.
3. The refund is best-effort — if the return invoice has expired or payment fails, the funds are retained by the server.

Servers SHOULD run periodic cleanup of expired sessions (e.g. hourly) rather than checking TTL on every request.

## Settlement

### Deposit Settlement

The deposit follows the standard Lightning settlement model — the client pays a BOLT11 invoice, and the preimage serves as proof of payment. See the [Lightning payment method](lightning.md) for details.

### Refund Settlement

On session close, the server pays the client's return invoice using the Lightning Network. This requires the server to have a `PaymentSender` capability (the ability to send outbound Lightning payments).

The return invoice MUST be amountless (no encoded amount). The server sets the payment amount to the computed refund. If the return invoice has expired by close time, the refund fails and the funds are retained.

## Security Considerations

### Bearer Token Reuse

The deposit preimage is reused as a bearer token for every request in the session. This is a weaker security model than per-request Lightning payments (where each preimage is single-use). Mitigations:

- **TLS required.** Servers and clients MUST use HTTPS.
- **Treat as session cookie.** Clients SHOULD store the preimage with the same care as a session cookie or API key.
- **Rate limiting.** Servers MAY implement per-session rate limiting to mitigate stolen-token abuse.
- **Short TTL.** Shorter session TTLs reduce the window of exposure.

### Return Invoice Validation

The server MUST validate that return invoices are amountless. If a client provides an invoice with an encoded amount, they could claim a refund larger than their remaining balance by providing an invoice for the full deposit amount. The server MUST reject session opens with amount-bearing return invoices.

### Session Isolation

Each session MUST be independently tracked. A bearer token for session A MUST NOT grant access to session B, even if both sessions are owned by the same client (same IP, same payment source).

### Atomic Balance Operations

All balance operations (debit, top-up, close) MUST be atomic. Non-atomic operations create race conditions where concurrent requests could overdraw the balance.

### Deposit Amount

Servers control the deposit amount and SHOULD set it proportional to expected usage (e.g. 20x the per-request cost). Excessively large deposits increase the client's counterparty risk. Excessively small deposits increase the frequency of top-ups.

## Method Compatibility

This specification defines the session intent with Lightning-specific flows (BOLT11 invoices, preimages, return invoices). The session lifecycle model (deposit → bearer → top-up → close) is method-agnostic. Future extensions could adapt it for other payment methods:

- **Cashu sessions:** Deposit via ecash token. Bearer via blinded token. Refund via fresh token issuance.
- **On-chain sessions:** Deposit via on-chain transaction. Refund via on-chain return.

Method-specific extensions SHOULD be defined in separate documents.

## Reference Implementations

| Implementation | Language | Notes |
|----------------|----------|-------|
| [@forgesworn/toll-booth](https://github.com/forgesworn/toll-booth) | TypeScript | Session intent planned. Charge intent implemented. |
| [Aperture PR #220](https://github.com/lightninglabs/aperture/pull/220) | Go | Session intent implemented (open, not yet merged). |
