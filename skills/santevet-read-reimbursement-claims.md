---
name: Read SantéVet reimbursement claims
description: >-
  Retrieve a pet-insurance reimbursement claim, list every claim for an insured animal or a
  client, and read the third-party-payment (tiers payant / PayVet) instalment schedule for a
  coverage. Read-only — this skill does not file claims and does not modify payment details.
api: openapi/santevet-reimbursement-openapi.yml
base: https://reimbursement.api.santevet.com
operations:
  - findReimbursement
  - findAllReimbursementsByAnimal
  - findAllReimbursementsByClient
  - findSchedule
generated: '2026-08-17'
method: generated
source: openapi/santevet-reimbursement-openapi.yml
---

# Read SantéVet reimbursement claims

This is the API behind SantéVet's claims settlement, including **PayVet** — third-party payment,
where SantéVet settles directly with the veterinary clinic instead of reimbursing the owner
afterwards.

## Before you start

- **The specification says nothing about authentication. It is wrong.** The document declares no
  `securitySchemes` and no `security` requirement, yet every operation returns
  `401 {"message":"User authentication required"}` anonymously. Send the partner API key in the
  `Authorization` header, matching the sibling Toolkit API's declared scheme.
- **All paths are under `/api/v1/`.** This is the only SantéVet API with a version in its path.
- **The error envelope is a flat `{"message": "..."}`.** It is not RFC 7807 and it differs from
  the `problem+json` and Hydra envelopes the Toolkit API returns. There is no error code and no
  stable error identifier to branch on — only the HTTP status and a human-readable string.
- **You are handling health and financial records about a named customer's animal.** Retrieve
  only what you were asked for.

## Steps

1. **Retrieve one claim.** `findReimbursement` — `GET /api/v1/reimbursements/{reimbursementId}`.
   Returns the `ApiReimbursement` schema, 25 properties, the richest projection available.
   Documented failures: `400` invalid identifier, `404` not found.

2. **List every claim for an animal.** `findAllReimbursementsByAnimal` —
   `GET /api/v1/animals/{animalId}/reimbursements`. Returns an array of `ApiReimbursement2`
   (18 properties). `404` when the animal does not exist.

3. **List every claim for a client.** `findAllReimbursementsByClient` —
   `GET /api/v1/clients/{clientId}/reimbursements`. Returns an array of `ApiReimbursement3`
   (19 properties). `404` when the client does not exist.

4. **Read the third-party-payment schedule.** `findSchedule` —
   `GET /api/v1/third-party-payments/{coverageId}/schedule`. Returns `ApiSchedule`, which
   contains `due_dates[]` of `ApiDueDate`. This is the instalment plan for a PayVet coverage.
   The only documented response is `200`; it declares no failure mode, so treat any non-`200`
   defensively.

## Reading the response

Three near-identical schemas exist and **the names do not tell you which is which**:

| Schema | Properties | Returned by |
|---|---|---|
| `ApiReimbursement` | 25 | `findReimbursement` |
| `ApiReimbursement2` | 18 | `findAllReimbursementsByAnimal` |
| `ApiReimbursement3` | 19 | `findAllReimbursementsByClient` |

The same applies to `ApiClinic` / `ApiClinic2` / `ApiClinic3`. Do not assume a field present on
one is present on another — a list projection carries fewer fields than the single-item
projection. If you need the full record, fetch it with `findReimbursement`.

Nested structures worth knowing: a reimbursement has `typologies[]` (`ApiTypology`), each
typology has `acts[]` (`ApiAct`) and one `upper_limit` (`ApiUpperLimit`).

## Identifiers you cannot follow

A reimbursement carries eight `*_id` fields, and **most of them resolve to nothing** in any
published SantéVet contract:

- `contract_id`, `clinic_id`, `veterinary_id`, `sinister_notification_id`, `correspondence_id` —
  **no operation exists** to resolve any of these. Report the raw value; do not claim to have
  looked it up.
- `coverage_id` — usable only as the path parameter of `findSchedule`.
- `animal_id` — usable only as the path parameter of `findAllReimbursementsByAnimal`.
- `origin_id` — *probably* the Toolkit API's `OrigineCommerciale` (`GET /commercial-origins/{id}`),
  by name and type agreement. **SantéVet does not document this binding.** If you follow it, say
  that the link is inferred.
- The `clientId` path parameter has no corresponding client resource anywhere in the estate — you
  must be given the identifier, you cannot discover it.

Claim classification vocabulary does live in the Toolkit API: `getTypeSinistreCollection`
(`GET /claims/types`) and `getMotifNonRemboursementCollection` (`GET /claims/no-refund-reasons`).
Use those to turn a numeric claim type or refusal reason into a label.

## Rules

- **Never call `createReimbursement`** (`POST /api/v1/reimbursements`). It files a real insurance
  claim, takes `multipart/form-data`, declares no failure response and has **no idempotency
  key** — a retry can file a duplicate claim against a customer's policy.
- **Never call `updateReimbursement`** (`PATCH /api/v1/reimbursements/{reimbursementId}`). Its
  request body is `UpdateThirdPartyPaymentRequest`, which carries `clinic_id` and
  `veterinary_id` — it redirects who gets paid. That is a payment-instruction change and requires
  a human.
- **Do not retry a `400`.** It means the identifier is malformed, not transient.
- **Distinguish `401` from `404`.** A `401` is your credential; a `404` is the record. Never
  report a `401` to a user as "no claims found".
- **There is no rate-limit header and no request-id header.** You get no runtime backoff signal
  and no correlation handle to quote to SantéVet support. Pace conservatively and log the full
  request yourself.
- **Never invent a reimbursement amount, status or due date.** If a field is absent from the
  projection you fetched, fetch the fuller one or say it is unavailable.
