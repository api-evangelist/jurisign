---
name: jurisign-bulk-and-public-forms
description: >-
  Send the same document to many recipients as a JuriSign bulk campaign, or publish a shareable public form that
  lets each respondent create and sign their own request without an account. Use for onboarding batches, policy
  acknowledgements, and open sign-up flows.
api: JuriSign REST API
base_url: https://www.jurisign.fr/api/v1
operations:
  - createBulkTemplate
  - listBulkTemplates
  - getBulkTemplate
  - updateBulkTemplate
  - deleteBulkTemplate
  - createBulkCampaign
  - launchBulkCampaign
  - cancelBulkCampaign
  - retryBulkCampaign
  - getBulkCampaign
  - exportBulkCampaignResults
  - deleteBulkCampaign
  - createPublicForm
  - getPublicForm
  - updatePublicForm
  - rotatePublicFormToken
  - deletePublicForm
  - listTemplates
  - createSignRequestFromTemplate
generated: '2026-08-31'
method: generated
source: openapi/jurisign-api-openapi.yml + https://www.jurisign.fr/developpeurs
---

# Many signatures at once

Two different shapes, and picking the wrong one costs a rewrite.

- **Bulk campaign** — *you* hold the recipient list. One personalised document per recipient, sent by you.
- **Public form** — *you don't*. You publish a link; whoever opens it creates and signs their own request, with no
  account.

Both require the `sign-requests:write` scope (which also covers the templates and bulk endpoints).

## Bulk campaigns

### 1. Bulk template — `createBulkTemplate`

`POST /bulk-templates` with `name`, `html_content` and `merge_fields`. The merge fields are the placeholders that
get personalised per recipient. JuriSign's own framing: recipients go in as JSON, so there is no Excel file to
generate.

Update it with `PUT /bulk-templates/{id}`; delete with `DELETE /bulk-templates/{id}`.

### 2. Campaign — `createBulkCampaign`

`POST /bulk-campaigns` referencing the bulk template and carrying the recipients. This creates the campaign; it
does not send anything.

### 3. Launch — `launchBulkCampaign`

`POST /bulk-campaigns/{id}/launch`.

**This is the highest-consequence call in the whole API.** It fans out to every recipient at once and spends one
signature credit per recipient. If you are acting unattended, get a human to approve this specific call, and check
the recipient count against `GET /bulk-campaigns/{id}` first.

Rehearse it with a `sandbox_` token. Nothing is delivered and nothing is charged, but the campaign mechanics and
the webhooks are real.

### 4. Watch, fix, export

- `GET /bulk-campaigns/{id}` — `status`, `progress` and a `stats` object.
- `POST /bulk-campaigns/{id}/cancel` — stop a running campaign.
- `POST /bulk-campaigns/{id}/retry` — resets recipients **in error** back to pending so the campaign can be
  launched again. It does not re-send to people who already received it.
- `GET /bulk-campaigns/{id}/export` — campaign results.
- `DELETE /bulk-campaigns/{id}` — **returns 422 while the campaign is processing.** Cancel first.

Subscribe to `signer.signed` and `sign_request.completed` rather than polling the campaign in a loop; regular
calls are capped at 60/minute per user.

## Public forms

### Publish — `createPublicForm`

`POST /public-forms`. A form is backed by a regular **template** (`listTemplates` / the `Template` resource), not
by a bulk template — that is the distinction people get wrong. The response carries `public_url`, the shareable
link, plus `expires_at` and `daily_submission_limit` if you set them.

Each respondent who opens the link gets their own signature request. Track uptake with `submissions_count` and
`last_submitted_at` on `GET /public-forms/{id}`, and subscribe to the `public_form.submitted` webhook event.

### If the link leaks — `rotatePublicFormToken`

`POST /public-forms/{id}/rotate-token` mints a new public token and **the previous link stops working
immediately**. JuriSign documents this as the remedy for a link that leaked or was shared too widely. Signatures
already collected through the old link are not undone — rotation closes the door, it does not reverse what came
through it.

Set `expires_at` and `daily_submission_limit` at creation rather than relying on rotation after the fact.

`DELETE /public-forms/{id}` removes the form entirely.

## One-off from a template

If you just want a single request from a saved template, skip both of the above:
`POST /templates/{id}/sign-requests` — `createSignRequestFromTemplate` — creates the whole request in one call
with the template's signers, zones and options already applied.

## Errors worth pre-empting

- `422` on `deleteBulkCampaign` — the campaign is processing. Cancel, then delete.
- `422` on campaign creation — validation; read the per-field messages under `errors`.
- `403` — the token is missing `sign-requests:write`, or the campaign belongs to another organization.
- `429` — 60 regular calls per minute per user. Honour `Retry-After`.
