# LNURLcash Payment Method

**Method identifier:** `lnurlcash`

**Registry:** HTTP Payment Methods (IANA)

**Parent specification:** [draft-ryan-httpauth-payment-01](https://datatracker.ietf.org/doc/draft-ryan-httpauth-payment/)

**Status:** Proposed

---

## Overview

The `lnurlcash` payment method enables HTTP Payment Authentication using [LUD-25](https://github.com/lnurl/luds) bearer notes. The server specifies accepted mints in the challenge; the client sends a note URL as the credential. The server rotates the note at the mint to verify its value, prevent double-spending, and take ownership.

A bearer note is a URL carrying a 32-byte secret. Whoever holds the secret holds the value, and value moves by moving the URL: there is no invoice to pay, no payment hash to wait on, and no on-chain footprint. Because a note is just a URL, it fits anywhere a string fits: a header, a QR code, an NFC tag, a chat message.

Settlement is a single call. Rotating a note replaces its secret with one only the server knows, and that one call is simultaneously the proof the note is live, the double-spend check, and the transfer of ownership. There is no separate redemption step and no replay table to keep, because a note presented twice is refused by the mint the second time.

## Challenge Request

The challenge request is a JSON object embedded (base64url-encoded, JCS-canonicalised) in the `request` parameter of the `WWW-Authenticate: Payment` header.

```http
WWW-Authenticate: Payment id="<hmac>",
    realm="api.example.com",
    method="lnurlcash",
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
    "mints": ["mint.example.com"],
    "mintPubkeys": ["02f1c3a9b7e5d4c2a0918273645546372819a0b1c2d3e4f5061728394a5b6c7d8e"]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | Yes | Amount in the smallest unit of `currency`. |
| `currency` | string | Yes | Currency code. MUST be `sat` (satoshis). Bearer notes are denominated in millisatoshis on the wire; see Note Value Validation. |
| `methodDetails.mints` | string[] | Yes | Array of accepted mints, each a host (`mint.example.com`) or any URL on that host. At least one. |
| `methodDetails.mintPubkeys` | string[] | No | When present, clients SHOULD only send signed notes, and servers MUST verify each note's signature against a pinned key. When absent, an unsigned note is acceptable. |

**JSON Schema:** [`schemas/lnurlcash-charge-request.json`](schemas/lnurlcash-charge-request.json)

## Credential Payload

The credential payload is included in the `payload` field of the `Authorization: Payment` credential.

```json
{
  "note": "lnurlw://mint.example.com/w?k1=9f2b7c1e4a6d8f0b3c5e7a9d1f3b5c7e9a1d3f5b7c9e1a3d5f7b9c1e3a5d7f9b&amount=21000"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `note` | string | Yes | Bearer note URL, in either the LUD-17 scheme form (`lnurlw://…`) or the fetchable form (`https://…`). MUST carry a `k1` query parameter holding a 32-byte hex secret. |

The note's mint MUST be one of the `mints` in the challenge request. Servers MUST reject notes issued anywhere else, and MUST do so before making any request to the note's host.

**JSON Schema:** [`schemas/lnurlcash-charge-payload.json`](schemas/lnurlcash-charge-payload.json)

## Verification Procedure

Verification requires **network calls to the mint**. It cannot be performed stateless.

1. Parse the `Authorization: Payment` header. Decode the base64url credential.
2. Verify the HMAC-bound challenge ID (per parent draft §4.3).
3. Check the challenge `expires` field. Reject if expired.
4. Extract the `note` from the credential payload.
5. Parse the note URL. It MUST carry a `k1` of exactly 32 bytes, hex-encoded.
6. Verify the note's host matches one of the `mints` in the challenge request. Do this **before** any network call: an unchecked host would let a caller aim the server's requests wherever it liked.
7. **Fetch the note's information** (LUD-03 `withdrawRequest`). The response carries the callback URL, the note's `maxWithdrawable` in millisatoshis, and the mint's `mintPubkey`. This value, not the `amount` in the URL, is what the note is worth.
8. If `mintPubkeys` was specified in the challenge, verify the note URL's `sig` against the pinned key over the value from step 7. Reject an unsigned note.
9. Verify `maxWithdrawable` >= the requested `amount` converted to millisatoshis.
10. **Rotate the note** (LUD-25). Generate a fresh 32-byte secret, send only its SHA-256 hash to the callback alongside the presented `k1`. This is the critical step. It atomically:
    - Verifies the note is live (an unknown or spent note is refused here)
    - Prevents double-spending (the mint burns the presented secret)
    - Transfers ownership (the new secret is known only to the server)
11. If all checks pass, the credential is valid.

**CRITICAL:** Servers MUST rotate the note before granting access. Accepting a note without rotating it (e.g. treating the informational GET in step 7 as sufficient, or deferring the rotate to a background job) creates a double-spend vulnerability: the same note can buy access from any number of servers at once. The rotate is the replay protection mechanism.

Servers MUST generate the replacement secret themselves and send only its hash. A mint that generates the replacement secret and hands it back has been a holder of the note, and the server's claim to it is not exclusive. A server that receives a secret it did not generate SHOULD treat the settlement as failed.

## Mint Failure Handling

The mint is an external dependency. When it is unreachable or returns an unexpected error:

- The server SHOULD respond with `503 Service Unavailable` with a `Retry-After` header.
- The server SHOULD NOT re-challenge with `402`, as this would cause the client to prepare a fresh note unnecessarily (an extra split, and an extra fee).
- If the error is clearly a client issue (malformed note, wrong mint, spent note, insufficient value), the server SHOULD respond with `402` and a fresh challenge.

A rotate whose response is lost is a distinct case: the mutation may have landed, which would burn the client's note without the server obtaining a usable replacement. Servers MUST NOT grant access on an unconfirmed rotate. Servers SHOULD retain the generated secret so the outcome can be established later, and SHOULD respond `503` rather than `402`, since the client's note may be gone.

## Settlement Procedure

The rotate IS settlement. Once the server has rotated the note, it holds a note of equivalent value under a secret only it knows, and the secret the client presented is burned.

No separate settlement step is required. The rotate is atomic and final.

Servers MAY choose what to do with the received note:
- **Hold:** Keep the note URL in the server's wallet.
- **Melt:** Spend it into a BOLT11 invoice on the server's own Lightning node (LUD-25 melt). The mint burns the note only once the outgoing payment settles; where the mint offers LUD-21 verification, the server can confirm before treating the funds as arrived.
- **Forward:** Send the note URL to another wallet or service. Rotate again first if the recipient should not be able to trace it to this payment.

## Security Considerations

### Double-Spend Prevention

The rotate (step 10 of verification) is the sole mechanism preventing double-spend. Servers MUST NOT skip it, defer it, or replace it with a local record of seen notes. A note that has not been rotated may be spent elsewhere simultaneously.

### Mint Trust

The server trusts the mint to honour the rotate, to report the note's value honestly, and to refuse a burned secret. Clients trust the mint to honour redemptions. Both parties should only use mints with established reputations. The `mints` field in the challenge allows servers to restrict accepted mints, and `mintPubkeys` allows them to pin a mint's key so a note's value carries a signature they can check.

### Privacy Properties

This method's privacy profile is the inverse of the [`cashu`](cashu.md) method's, and the difference is worth stating plainly.

Against the mint, this method is weaker. Notes are not blinded. The mint sees every rotate, so it knows a note changed hands, when, and which note replaced it. A mint can therefore follow a note through its holders, and can tell that the server rotating a note is not the party it issued the note to.

Against the server, this method is comparable. The server learns nothing about the payer that the request itself did not already carry: there is no identity, no invoice, and no payment hash tying the payer to anything outside this exchange. A note presented as payment says only that its holder had it.

Choose accordingly. Where the mint is the adversary, `cashu` is the better method. Where the server is the adversary, or where the value must survive as a URL that can be handed over out of band, `lnurlcash` costs nothing in privacy against the party being paid.

### Transport Security

Servers and clients MUST use TLS (HTTPS). Bearer notes are bearer instruments, and anyone who intercepts a note can spend it.

A note travels in a request header, which makes it more exposed than a body field. Servers MUST NOT write the credential payload to access logs, error reports, or traces, and operators SHOULD check that reverse proxies and observability tooling in front of the server do not capture request headers wholesale. A note in a log file is money in a log file.

### Note Value Validation

Notes are denominated in millisatoshis, and the challenge `amount` is in satoshis. Servers MUST compare against the value reported by the mint in step 7, converted consistently, and MUST NOT trust the `amount` parameter in the note URL, which is a hint written by whoever last held the note.

Servers MUST verify that the note's value is >= the requested amount. Notes may be worth more than required. Servers SHOULD NOT return change. The client is responsible for presenting a note of appropriate denomination, splitting one at the mint beforehand if necessary. If overpayment occurs, the excess is retained by the server.

### Secret Handling

The `k1` in a note URL is the value. Servers MUST generate replacement secrets with a cryptographically secure random source, MUST NOT derive them from anything the client supplied, and MUST NOT reuse one across settlements. A replacement secret is live money the moment the rotate lands, so it MUST be persisted before it can be lost to a crash if the server intends to keep the note.

## Protocol References

| LUD | Title | Relevance |
|-----|-------|-----------|
| [LUD-25](https://github.com/lnurl/luds) | LNURLcash bearer notes | Note format, rotate, split, merge and melt |
| [LUD-03](https://github.com/lnurl/luds/blob/luds/03.md) | `withdrawRequest` | The informational GET that reports a note's value and callback |
| [LUD-17](https://github.com/lnurl/luds/blob/luds/17.md) | Protocol schemes | The `lnurlw://` spelling of a note URL |
| [LUD-21](https://github.com/lnurl/luds/blob/luds/21.md) | `verify` | Confirming that a melt's outgoing payment settled |
| [BOLT11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md) | Invoice format | Melting a note onto the server's own Lightning node |

## Reference Implementations

| Implementation | Language | Notes |
|----------------|----------|-------|
| [@forgesworn/toll-booth](https://github.com/forgesworn/toll-booth) | TypeScript | lnurlcash rail (`X-LNURLcash` header scheme). IETF Payment adapter planned. |
