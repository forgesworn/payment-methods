# Payment Methods & Intents for HTTP Payment Authentication

This repository defines payment method and intent specifications for the IETF `draft-ryan-httpauth-payment` HTTP payment authentication scheme. It is a specification repository (Markdown specs plus JSON Schema), not a code library: there is no build, test or lint step and no package manifest.

## Structure

```
README.md        overview, registry table, how-it-works flow
lightning.md      "lightning" payment method spec
cashu.md          "cashu" payment method spec
lnurlcash.md       "lnurlcash" payment method spec
session.md         "session" payment intent spec
schemas/          JSON Schema for each method/intent's wire formats
SECURITY.md       security policy
```

## Conventions

- British English in prose.
- Amounts are expressed in the smallest currency unit (satoshis).
- Each payment method or intent has a matching pair of JSON Schema files in `schemas/` (charge/challenge request and credential/payload), named `<method>-charge-request.json` and `<method>-charge-payload.json` (or `<intent>-<step>-*.json` for the session intent).
- Specification identifiers (`lightning`, `cashu`, `lnurlcash`, `session`) are lowercase and stable: do not rename them without updating the README registry table and every schema `$id`.

## Common Pitfalls

- This is a specification, not an implementation: do not add runtime code or dependencies here. Reference implementations live in separate repositories (see the README's "Reference Implementations" table).
- Schema `$id` values are absolute GitHub URLs; keep them consistent with the file's location if a schema is renamed or moved.
- The "Publication Roadmap" in the README describes IETF/IANA submission status; do not describe registries as final or ratified.

## How to Verify a Change

- Check that any schema edit is valid JSON Schema (draft-07) and that its `$id` and filename stay in sync.
- Check that the README registry table, the spec file it points to and the schema pair it references stay consistent.
