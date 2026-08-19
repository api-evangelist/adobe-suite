---
name: Extract structured content from a PDF with Adobe PDF Services
description: >-
  Authenticate against Adobe IMS, upload a PDF to Adobe-managed storage, run the Extract
  operation, poll the job to completion, and download the structured JSON result.
api: openapi/adobe-suite-pdf-services-openapi.json
generated: '2026-08-13'
method: generated
source: openapi/adobe-suite-pdf-services-openapi.json (49 operations, verified operationIds)
operations:
  - authentication.generatetoken
  - asset.uploadpresignedurl
  - pdfoperations.extractpdf
  - pdfoperations.extractpdf.jobstatus
  - asset.get
  - asset.delete
---

# Extract structured content from a PDF

Base URL: `https://pdf-services-ue1.adobe.io` (US East) or `https://pdf-services-ew1.adobe.io` (EU West).
Pick the region that matches your data-residency requirement; they are not interchangeable.

## 1. Get a token

Call `authentication.generatetoken` (`POST /token`) with your Client ID and Client Secret
from the Adobe Developer Console. The token is an Adobe IMS access token and is valid for
**24 hours**. Every subsequent call needs both:

- `Authorization: Bearer <token>`
- `x-api-key: <Client ID>`

Do not mint a new token per request — you will burn rate budget for nothing.

## 2. Upload the PDF

Call `asset.uploadpresignedurl` (`POST /assets`) with `mediaType: application/pdf`.
It returns an `assetID` and a pre-signed `uploadUri`. `PUT` the file bytes directly to
`uploadUri` — that upload does **not** go through the Adobe API host and does not carry
your Authorization header.

Maximum file size is **100MB**.

## 3. Submit the extract job

Call `pdfoperations.extractpdf` (`POST /operation/extractpdf`) with the `assetID` and the
elements you want (`elementsToExtract: ["text","tables"]`, optionally
`elementsToExtractRenditions`). This is asynchronous: it returns **201** with a `location`
header, not a result.

## 4. Poll to completion

Call `pdfoperations.extractpdf.jobstatus` (`GET /operation/extractpdf/{jobID}/status`)
until `status` is `done` or `failed`. Every Adobe creative and document API uses this
submit-then-poll shape — see `conventions/adobe-suite-conventions.yml`.

Back off between polls. There is no `X-RateLimit-*` header to read: your only signals are
**429 Too Many Requests** with `Retry-After`, and the Free Tier ceiling of **25 requests
per minute** (Enterprise: 100 RPM). See `rate-limits/adobe-suite-rate-limits.yml`.

## 5. Download and clean up

The terminal status response carries a download URI for the result asset. Fetch it, then
call `asset.delete` (`DELETE /assets/{assetID}`) on both the input and output assets.
Adobe expires assets automatically, but deleting is the polite and auditable behaviour.

## Cost and error rules

- Every submitted operation consumes a **Document Transaction**. The Free Tier is 500 per
  month. A retried POST costs another transaction — **there is no idempotency key on any
  PDF Services operation** (`conventions/adobe-suite-conventions.yml`). Never blind-retry
  a `pdfoperations.*` POST; poll the existing job instead.
- `429` → back off, honour `Retry-After`.
- `4xx` other than 429 → do not retry; the request is wrong, not unlucky.
- Errors come back as `{"error": {"code": "...", "message": "..."}}` — **not** RFC 9457
  problem+json. See `errors/adobe-suite-problem-types.yml`.
- Quote the `x-request-id` from the response when opening a support ticket.

## Related operations in the same contract

`pdfoperations.pdftomarkdown` (PDF → markdown, the LLM-friendly output),
`pdfoperations.createpdf`, `pdfoperations.htmltopdf`, `pdfoperations.combinepdf`,
`pdfoperations.compresspdf`, `pdfoperations.exportpdf`, `pdfoperations.pdftoimages`,
`pdfoperations.linearizepdf`, `pdfoperations.ocr`, and
`pdfoperations.documentgeneration`. Each has a matching `.jobstatus` operation and
follows the identical five-step shape above.
