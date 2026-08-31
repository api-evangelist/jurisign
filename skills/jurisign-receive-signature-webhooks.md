---
name: jurisign-receive-signature-webhooks
description: >-
  Register a JuriSign webhook endpoint, verify its HMAC signature, handle the twelve signature and payment events,
  and diagnose failed deliveries from the endpoint's own delivery log. Use instead of polling sign-request status.
api: JuriSign REST API
base_url: https://www.jurisign.fr/api/v1
operations:
  - createWebhook
  - listWebhooks
  - updateWebhook
  - deleteWebhook
  - getWebhookLogs
  - regenerateWebhookSecret
generated: '2026-08-31'
method: generated
source: openapi/jurisign-api-openapi.yml + https://www.jurisign.fr/developpeurs
---

# Receive signature events

Webhooks are first-class resources here — they have CRUD, a delivery log and a rotatable secret. Requires the
`webhooks:manage` scope. Note that webhooks are not available on the free Découverte plan or the Pack 100.

## Register — `createWebhook`

`POST /webhooks` with an HTTPS `url` (max 500 characters) and an `events` array with at least one entry.

**The 201 response contains the signing secret, and it is shown exactly once.** Store it before you do anything
else. There is no endpoint that will show it to you again — only `POST /webhooks/{id}/regenerate-secret`, which
mints a new one and invalidates the old immediately.

## The twelve events

The enum is closed in the OpenAPI. Subscribe to what you handle, not to everything.

Signature request lifecycle:
- `sign_request.sent` — sent to signers
- `sign_request.completed` — every signer has signed
- `sign_request.cancelled` — cancelled
- `sign_request.relaunched` — reminded / relaunched

Approval workflow:
- `sign_request.submitted_for_approval`
- `sign_request.approved`
- `sign_request.rejected`

Individual signers:
- `signer.signed`
- `signer.declined`

Public forms:
- `public_form.submitted` — a respondent submitted a shareable self-signing form

Pay-at-signature:
- `payment.completed`
- `payment.failed`

For the common "tell me when it's done" case, `sign_request.completed` plus `signer.signed` is enough.

## Verify the signature

Deliveries are signed HMAC-SHA256 with the secret from creation. Compute the HMAC over the raw request body and
compare in constant time. Reject anything that does not match — do not parse it first.

**Header name: JuriSign publishes two.** The developer page (current, matching the v3.24.1 era) documents
`X-Jurisign-Signature: sha256=<hex>`. The older integration guide says `X-Signature`. The header is not in the
OpenAPI, so neither can be confirmed from the contract. Read whichever of the two is present on the first real
delivery you receive, log which one it was, and code against that. Do not guess.

## Retries

A failed delivery is retried 8 times over roughly 42 hours, so an outage on your side does not lose the event.
(The older integration guide still describes an earlier 3-attempt policy at 1, 5 and 30 minutes. The 8-attempt
figure is the one the provider markets and it matches the `attempts` and `next_retry_at` fields on the delivery
log.)

Design your receiver accordingly:

- Return 2xx fast. Acknowledge, then do the work asynchronously.
- Expect duplicates. Deliveries are at-least-once — key your handling on the sign-request or signer id plus the
  event name, and make replays a no-op.
- Never make the signature check depend on anything slow.

## Diagnose — `getWebhookLogs`

`GET /webhooks/{id}/logs` returns per-attempt records: `event`, `status` (`success` / `pending` / `failed`),
`response_status` (the code *your* server returned), `attempts`, `next_retry_at`, `created_at`. The endpoint
resource itself carries a `stats` object with `success`, `failed` and `pending` counts.

If `response_status` is a 5xx from your side, fix the receiver and the outstanding attempts will still arrive
within the retry window. If it is null, the delivery never connected.

## Rotate the secret — `regenerateWebhookSecret`

`POST /webhooks/{id}/regenerate-secret`. The old secret stops working immediately, so there is no overlap window:
deploy the new secret to your receiver in the same change, or accept a brief gap of rejected deliveries that the
retry policy will re-deliver.

## Test it for real

Webhooks are delivered for real even with a `sandbox_` token — that is a deliberate design decision by JuriSign,
on the grounds that a webhook you never receive proves nothing. So the full sandbox flow (upload → create → send →
sign) exercises your receiver end to end without sending anything to a real signer or spending a credit.

## Rate limit

Webhook management calls are limited to 30 per minute per authenticated user. Exhaustion returns 429 with
`Retry-After`.
