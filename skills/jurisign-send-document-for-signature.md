---
name: jurisign-send-document-for-signature
description: >-
  Upload a PDF to JuriSign and get it signed by one or more people with email or SMS OTP verification, then
  download the signed document and its audit proof. Use for any single-document eIDAS SES signature flow.
api: JuriSign REST API
base_url: https://www.jurisign.fr/api/v1
operations:
  - createSandboxToken
  - createToken
  - uploadDocument
  - createSignRequest
  - sendSignRequest
  - getSignRequest
  - cancelSignRequest
  - downloadSignedPdf
  - downloadProof
generated: '2026-08-31'
method: generated
source: openapi/jurisign-api-openapi.yml + https://www.jurisign.fr/api/guide
---

# Send a document for signature

Three calls do the work. A fourth watches it finish.

## Before you start

Rehearse in sandbox. `POST /auth/sandbox-token` with the same email and password you would use for
`POST /auth/token` returns a token prefixed `sandbox_`. Every response is identical to production, no email or SMS
reaches anyone, no signature credit is spent, no invoice is raised — and webhooks still fire, deliberately, because
a webhook you never receive proves nothing. Sandbox and live records are isolated from each other; a sandbox token
used without its prefix, or a live token used against sandbox, returns 401.

Pass the token as `Authorization: Bearer {token}` on every call below.

## 1. Upload the document — `uploadDocument`

`POST /documents`, `multipart/form-data`, 20 MB maximum.

- One file: send `file`.
- A contract plus annexes: send `files[]` — up to 10 PDFs, images or Word documents. JuriSign merges them into one
  PDF **in send order** and returns `merged_from` with the count actually merged. Check it matches what you sent.

Keep `data.id`. You need it in step 2. Also note `data.page_count` — signature zones are placed by page number.

## 2. Create the signature request — `createSignRequest`

`POST /sign-requests`, JSON.

**Send an `Idempotency-Key` header.** Use your own order or case reference. It is organization-scoped and honoured
for 24 hours. If the network drops after JuriSign received the call but before you saw the response, replay the
exact same call with the same key: you get the original response back with `Idempotent-Replayed: true`, and no
second request is created — so no second SMS is billed. Reusing the key with a *different* body returns 409; that
is an integration bug, not a retry. Failed responses never consume a key.

Body essentials:

- `document_id` — from step 1.
- `subject` — required.
- `signers[]` — each needs `prenom`, `nom`, `email`. Set `otp_channel` to `"email"` (included everywhere) or
  `"sms"`, and when it is `"sms"` add `telephone` in international format (`+33612345678`).
- `signing_order_type` — `0` for simultaneous, `1` for sequential.
- `expiry_hours` — 24 to 720.
- `zones[]` — optional. `signer_index` binds a zone to a signer by position. `page`, `x`, `y`, `width`, `height`
  are **percentages of the page, 0–100, origin top-left** — not pixels.
- `redirect_url` — optional; returns the signer to your app afterwards. The domain must be registered with
  JuriSign beforehand.
- `auto_send: true` — optional; collapses step 3 into this call.

**The request is created in `draft` and nobody is notified.** This is the mistake JuriSign flags most often in its
own guide. Nothing has been spent and nothing has been sent yet.

## 3. Send it — `sendSignRequest`

`POST /sign-requests/{id}/send`.

This is the consequential call. It spends a signature credit, emails or SMSes real people, and moves the status
from `draft` to `pending`. If you are acting on someone's behalf without supervision, this is the point to stop and
confirm.

Skip this call only if you set `auto_send: true` in step 2.

## 4. Track and collect — `getSignRequest`, `downloadSignedPdf`, `downloadProof`

Prefer webhooks over polling — see the `jurisign-receive-signature-webhooks` skill. If you must poll,
`GET /sign-requests/{id}` returns `status` and `progress` plus per-signer state.

Statuses: `draft`, `pending`, `partially_signed`, `completed`, `expired`, `cancelled`.
Per-signer statuses: `pending`, `notified`, `signed`, `declined`.

Once `completed`:

- `GET /sign-requests/{id}/download` returns the signed PDF (`application/pdf`).
- `GET /sign-requests/{id}/proof` returns the audit proof PDF — timestamps, IP addresses, OTP records and SHA-256
  fingerprints. Store this alongside the signed document; it is what makes the signature defensible.

## Backing out

- Before send: nothing to undo. A draft request notifies no one.
- After send: `POST /sign-requests/{id}/cancel`, and **only while status is `pending`**. Once every signer has
  signed the request is `completed` and there is no cancel path — the signature is final and the proof exists.
- The uploaded document itself can be deleted only while it is still `draft`.

## When it refuses

Errors are always JSON, never problem+json. Shape is `{"message": "...", "errors": {"field.path": [...]}}`.

- `401` — token missing, invalid, revoked, or the wrong mode (sandbox vs live).
- `403` — the token lacks the required scope, or the resource belongs to another organization.
- `409` — `Idempotency-Key` reused with a different body.
- `422` — validation. Read each field message under `errors`; array members are addressed by index, e.g.
  `signers.0.email`. Also returned for state preconditions — deleting a non-draft document, cancelling a request
  that is not pending.
- `429` — rate limit. Uploads are 20/minute, send/cancel 30/minute, everything else 60/minute, all per
  authenticated user. **Wait for the `Retry-After` header** rather than retrying immediately. There are no
  `X-RateLimit-*` headers to read ahead.

## Least privilege

`POST /auth/token` accepts a `scopes` array. For this flow request only `documents:write` and
`sign-requests:write` (plus the matching `:read` scopes to collect the output). Omitting `scopes` grants all five,
including `webhooks:manage`, which this flow does not need.
