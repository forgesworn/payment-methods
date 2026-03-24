# Cashu Payment Method

**Method identifier:** `cashu`

**Registry:** HTTP Payment Methods (IANA)

**Parent specification:** [draft-ryan-httpauth-payment-01](https://datatracker.ietf.org/doc/draft-ryan-httpauth-payment/)

**Status:** Proposed

---

## Overview

The `cashu` payment method enables HTTP Payment Authentication using [Cashu](https://cashu.space) ecash tokens. The server specifies accepted mints in the challenge; the client sends a Cashu token as the credential. The server swaps the token's proofs at the mint to verify value and prevent double-spending.

Cashu ecash provides the strongest privacy of any payment method in this registry. Tokens use blind signatures — the mint cannot link the payer to the payment. There is no invoice, no payment hash, and no on-chain footprint. This makes the `cashu` method ideal for privacy-preserving machine payments and anonymous API access.

## Challenge Request

The challenge request is a JSON object embedded (base64url-encoded, JCS-canonicalised) in the `request` parameter of the `WWW-Authenticate: Payment` header.

```http
WWW-Authenticate: Payment id="<hmac>",
    realm="api.example.com",
    method="cashu",
    intent="charge",
    request="<base64url of charge request below>",
    expires="2026-03-24T12:15:00Z"
```

### Charge Request Schema

```json
{
  "amount": "100",
  "currency": "sat",
  "methodDetails": {
    "mints": ["https://mint.example.com"],
    "keysetIds": ["009a1f293253e41e"]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | Yes | Amount in the smallest unit of `currency`. |
| `currency` | string | Yes | Currency code. MUST correspond to a Cashu NUT-00 unit: `sat` (satoshis), `usd` (cents), `eur` (cents). |
| `methodDetails.mints` | string[] | Yes | Array of accepted Cashu mint URLs. At least one. |
| `methodDetails.keysetIds` | string[] | No | When present, clients SHOULD only send tokens using these keyset IDs. When absent, any active keyset from the accepted mints is valid. |

**JSON Schema:** [`schemas/cashu-charge-request.json`](schemas/cashu-charge-request.json)

## Credential Payload

The credential payload is included in the `payload` field of the `Authorization: Payment` credential.

```json
{
  "token": "cashuBo2F0gaJha..."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | string | Yes | Cashu token string (NUT-00 v4 format, `cashuA` or `cashuB` prefix). |

The token's unit MUST match the challenge `currency`. Servers MUST reject tokens whose unit does not match.

**JSON Schema:** [`schemas/cashu-charge-payload.json`](schemas/cashu-charge-payload.json)

## Verification Procedure

Verification requires a **network call to the Cashu mint**. It cannot be performed stateless.

1. Parse the `Authorization: Payment` header. Decode the base64url credential.
2. Verify the HMAC-bound challenge ID (per parent draft §4.3).
3. Check the challenge `expires` field. Reject if expired.
4. Extract the `token` from the credential payload.
5. Decode the Cashu token (NUT-00 v4 format).
6. Verify the token's mint URL matches one of the `mints` in the challenge request.
7. If `keysetIds` was specified in the challenge, verify the token's keyset ID is in the list.
8. **Swap proofs at the mint** (NUT-03). This is the critical step — it atomically:
   - Verifies the proofs are valid (correct blind signatures)
   - Prevents double-spending (the mint invalidates the old proofs)
   - Issues fresh proofs to the server
9. Verify the total swapped value is >= the requested `amount`.
10. If all checks pass, the credential is valid.

**CRITICAL:** Servers MUST swap tokens at the mint before granting access. Accepting tokens without swapping (e.g. for performance optimisation) creates a double-spend vulnerability. The swap is the replay protection mechanism.

## Mint Failure Handling

The Cashu mint is an external dependency. When it is unreachable or returns an unexpected error:

- The server SHOULD respond with `503 Service Unavailable` with a `Retry-After` header.
- The server SHOULD NOT re-challenge with `402`, as this would cause the client to obtain new tokens unnecessarily (wasting ecash).
- If the error is clearly a client issue (malformed token, wrong mint), the server SHOULD respond with `402` and a fresh challenge.

## Settlement Procedure

The proof swap IS settlement. Once the server successfully swaps the token's proofs at the mint, it holds fresh proofs of equivalent value. The old proofs are invalidated by the mint.

No separate settlement step is required. The swap is atomic and final.

Servers MAY choose what to do with received proofs:
- **Hold:** Keep proofs in the server's Cashu wallet.
- **Melt:** Convert proofs to Lightning by paying an invoice (NUT-05).
- **Forward:** Send proofs to another wallet or service.

## Security Considerations

### Double-Spend Prevention

The mint swap (step 8 of verification) is the sole mechanism preventing double-spend. Servers MUST NOT skip this step. A token that has not been swapped may be spent elsewhere simultaneously.

### Mint Trust

The server trusts the mint to correctly validate proofs and prevent double-spending. Clients trust the mint to honour redemptions. Both parties should only use mints with established reputations. The `mints` field in the challenge allows servers to restrict accepted mints.

### Privacy Properties

Cashu tokens use blind signatures (NUT-00). The mint signs tokens without seeing their final form. This means:
- The mint cannot link a specific token to the client who minted it.
- The server cannot determine the client's identity from the token.
- No payment hash or invoice links the transaction to an identity.

This provides stronger privacy than Lightning (where routing nodes see payment hashes) or on-chain methods (where transactions are publicly visible).

### Transport Security

Servers and clients MUST use TLS (HTTPS). Cashu tokens are bearer instruments — anyone who intercepts a token can spend it.

### Token Value Validation

Servers MUST verify that the total token value is >= the requested amount. Tokens may contain more value than required. Servers SHOULD NOT return change — the client is responsible for sending tokens of appropriate denomination. If overpayment occurs, the excess is retained by the server.

## Protocol References

| NUT | Title | Relevance |
|-----|-------|-----------|
| [NUT-00](https://github.com/cashubtc/nuts/blob/main/00.md) | Token format | Token encoding and decoding |
| [NUT-03](https://github.com/cashubtc/nuts/blob/main/03.md) | Swap | Proof swap for verification and double-spend prevention |
| [NUT-05](https://github.com/cashubtc/nuts/blob/main/05.md) | Melt | Converting proofs to Lightning (optional server-side) |
| [NUT-11](https://github.com/cashubtc/nuts/blob/main/11.md) | P2PK | Spending conditions on proofs (optional) |
| [NUT-18](https://github.com/cashubtc/nuts/blob/main/18.md) | Payment requests | Structured payment request format |

## Reference Implementations

| Implementation | Language | Notes |
|----------------|----------|-------|
| [@forgesworn/toll-booth](https://github.com/forgesworn/toll-booth) | TypeScript | xcashu rail (X-Cashu header scheme). IETF Payment adapter planned. |
