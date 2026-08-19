---
name: Build and export an audience in Adobe Experience Platform
description: >-
  Create a segment definition, preview its size, run a segment job, assemble it into an
  audience, and export the members — using the Segmentation Service API against a named
  sandbox.
api: openapi/adobe-suite-aep-segmentation-openapi.yaml
generated: '2026-08-13'
method: generated
source: openapi/adobe-suite-aep-segmentation-openapi.yaml (35 operations, verified operationIds)
operations:
  - createPreview
  - retrievePreview
  - createSegmentDefinition
  - createSegmentJob
  - retrieveSegmentJob
  - createAudience
  - listAudiences
  - createExportJob
  - retrieveExportJob
---

# Build and export an AEP audience

Base URL: `https://platform.adobe.io/data/core/ups` (the specs templatize this as
`https://{environment}.adobe.io/data/core/ups`).

## Headers — all four are required

Experience Platform is the strictest auth surface in the Adobe estate:

- `Authorization: Bearer <IMS token>`
- `x-api-key: <Client ID>`
- `x-gw-ims-org-id: <IMS Org ID>` — the tenant
- `x-sandbox-name: <sandbox>` — **the environment**

Get the sandbox name wrong and you will read or write the wrong tenant's data with a
200 response. Enumerate sandboxes first via the Sandbox API
(`openapi/adobe-suite-aep-sandbox-openapi.yaml`). See
`sandbox/adobe-suite-sandbox.yml`.

## 1. Preview before you commit

`createPreview` (`POST /preview`) with your PQL expression, then `retrievePreview`
(`GET /preview/{PREVIEW_ID}`) for the estimated qualifying profile count. Do this before
creating anything persistent — a bad PQL predicate that matches every profile is expensive
downstream.

## 2. Create the segment definition

`createSegmentDefinition` (`POST /segment/definitions`) with the PQL expression, schema
class, and evaluation method (`batch` or `streaming`).

Retrieve with `retrieveSegmentDefinitionById`, amend with `patchSegmentDefinition`,
remove with `deleteSegmentDefinition`. Use `bulkGetSegmentDefinitions`
(`POST /segment/definitions/bulk-get`) to fetch many by id — it is the correct call when
you would otherwise loop, and it keeps you inside the rate budget.

## 3. Evaluate it

`createSegmentJob` (`POST /segment/jobs`) with the segment ids, then poll
`retrieveSegmentJob` (`GET /segment/jobs/{SEGMENT_JOB_ID}`) until the job reports
`SUCCEEDED`. Batch segment jobs are minutes-to-hours, not seconds — poll on a long
interval and never tight-loop.

Abandon with `deleteSegmentJob`.

## 4. Register it as an audience

`createAudience` (`POST /audiences`), then `listAudiences` / `getAudience` to read it back.
Audiences are the object destinations and Journey Optimizer consume; segment definitions
are not.

## 5. Export the members

`createExportJob` (`POST /export/jobs`) naming the target dataset and the fields to
include, then `retrieveExportJob` (`GET /export/jobs/{EXPORT_JOB_ID}`). Cancel with
`cancelExportJob`.

## Rules

- **Pagination**: this API is cursor-based via `_links.next`. Do not assume page numbers —
  other Adobe products use offsets, this one does not. See
  `conventions/adobe-suite-conventions.yml`.
- **No published rate limit.** AEP documents 429 responses in its contracts but publishes
  no number. Treat 429 as authoritative, honour `Retry-After`, and keep concurrency low.
- **No idempotency key.** A retried `createSegmentJob` starts a second evaluation.
- Errors are not RFC 9457; the AEP specs declare `ErrorResponse` and `ProblemDetail`
  schemas inconsistently across services.

## Neighbouring contracts

26 other Experience Platform APIs are harvested in `openapi/adobe-suite-aep-*.yaml` —
schema registry, catalog, profile, query service, flow service, destinations, identity,
privacy, policy, data hygiene, sandbox and more. All share the four headers above.
