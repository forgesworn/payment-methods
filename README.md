# Payment Methods & Intents for HTTP Payment Authentication

Payment method and intent specifications for the IETF HTTP Payment Authentication scheme ([`draft-ryan-httpauth-payment-01`](https://datatracker.ietf.org/doc/draft-ryan-httpauth-payment/)).

The IETF draft defines a payment-method-agnostic `WWW-Authenticate: Payment` challenge-response flow. It establishes two IANA registries — **HTTP Payment Methods** and **HTTP Payment Intents** — for registering concrete payment mechanisms. This repository contains the first proposed specifications for those registries.

## Specifications

| Identifier | Registry | Specification | Status |
|------------|----------|---------------|--------|
| `lightning` | Payment Methods | [lightning.md](lightning.md) | Proposed |
| `cashu` | Payment Methods | [cashu.md](cashu.md) | Proposed |
| `session` | Payment Intents | [session.md](session.md) | Proposed |

### Lightning

Pay with the Bitcoin Lightning Network. The server issues a BOLT11 invoice; the client pays it and presents the preimage as proof. Verification is a single `SHA256(preimage) == paymentHash` comparison — stateless, no network call.

### Cashu

Pay with [Cashu](https://cashu.space) ecash tokens. The strongest privacy of any payment method — blind signatures mean the mint cannot link payer to payment. The server swaps token proofs at the mint for atomic verification and double-spend prevention.

### Session

Deposit-based prepaid access for streaming and metered use cases. The client deposits funds, uses a bearer token for subsequent requests, tops up when the balance runs low, and closes the session to receive a refund. Designed for LLM token streaming, ridesharing metering, and real-time API consumption.

## How It Works

```
Client                                          Server
  │                                                │
  │  GET /v1/chat/completions                      │
  │ ─────────────────────────────────────────────►  │
  │                                                │
  │  402 Payment Required                          │
  │  WWW-Authenticate: Payment method="lightning", │
  │    intent="charge", request="<invoice>", ...   │
  │  ◄─────────────────────────────────────────────│
  │                                                │
  │  [Client pays Lightning invoice]               │
  │                                                │
  │  GET /v1/chat/completions                      │
  │  Authorization: Payment <credential+preimage>  │
  │ ─────────────────────────────────────────────►  │
  │                                                │
  │  200 OK                                        │
  │  Payment-Receipt: <receipt>                    │
  │  {"choices": [...]}                            │
  │  ◄─────────────────────────────────────────────│
```

## JSON Schemas

Machine-readable schemas for all wire formats are in [`schemas/`](schemas/).

| Schema | Validates |
|--------|-----------|
| [`lightning-charge-request.json`](schemas/lightning-charge-request.json) | Lightning challenge request |
| [`lightning-charge-payload.json`](schemas/lightning-charge-payload.json) | Lightning credential payload |
| [`cashu-charge-request.json`](schemas/cashu-charge-request.json) | Cashu challenge request |
| [`cashu-charge-payload.json`](schemas/cashu-charge-payload.json) | Cashu credential payload |
| [`session-challenge-request.json`](schemas/session-challenge-request.json) | Session challenge request |
| [`session-open-payload.json`](schemas/session-open-payload.json) | Session open payload |
| [`session-bearer-payload.json`](schemas/session-bearer-payload.json) | Session bearer payload |
| [`session-topup-payload.json`](schemas/session-topup-payload.json) | Session top-up payload |
| [`session-close-payload.json`](schemas/session-close-payload.json) | Session close payload |

## Reference Implementations

| Implementation | Language | Methods | Intents |
|----------------|----------|---------|---------|
| [@forgesworn/toll-booth](https://github.com/forgesworn/toll-booth) | TypeScript | `lightning` (charge), `cashu` (adapted) | `session` (planned) |
| [Aperture PR #220](https://github.com/lightninglabs/aperture/pull/220) | Go | `lightning` (charge) | `session` (implemented) |

### Live Demo

[satgate](https://github.com/TheCryptoDonkey/satgate) — Lightning-gated AI inference at `satgate.trotters.dev`. Serves dual-scheme challenges (`L402` + `Payment`) on every 402 response.

```bash
# See the dual-scheme challenge
curl -s -D - https://satgate.trotters.dev/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"llama3","messages":[{"role":"user","content":"hello"}]}' \
  2>&1 | head -20
```

## Relationship to Other Standards

| Standard | Relationship |
|----------|-------------|
| [IETF `draft-ryan-httpauth-payment`](https://datatracker.ietf.org/doc/draft-ryan-httpauth-payment/) | Parent specification. Defines the `Payment` auth scheme and IANA registries. |
| [L402](https://docs.lightning.engineering/the-lightning-network/l402) | Lightning Labs' macaroon-based HTTP 402 protocol. Complementary — L402 uses reusable bearer tokens; IETF Payment is per-request. |
| [x402](https://www.x402.org/) | Coinbase's stablecoin payment protocol. Alternative payment method that could be registered in the same IETF registry. |
| [Cashu NUTs](https://github.com/cashubtc/nuts) | Cashu protocol specifications. Referenced by the `cashu` payment method. |
| [BOLT11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md) | Lightning invoice format. Used by the `lightning` method and `session` intent. |

## Contributing

These specifications are open for review and feedback. Please [open an issue](https://github.com/forgesworn/payment-methods/issues) for questions, suggestions, or implementation reports.

## Publication Roadmap

1. **Current:** GitHub specifications (this repository)
2. **Next:** Submit as IETF Internet-Drafts via [datatracker.ietf.org](https://datatracker.ietf.org)
3. **Goal:** IANA registration of `lightning`, `cashu`, and `session` identifiers

## Licence

[MIT](LICENSE)

---

Published by [ForgeSworn](https://github.com/forgesworn)
