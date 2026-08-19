---
name: Automate a Photoshop edit with the Adobe Photoshop API
description: >-
  Run a Photoshop server-side edit — remove background, create a mask, replace a smart
  object, or render a PSD — by submitting an async job and polling the matching status
  operation.
api: openapi/adobe-suite-photoshop-openapi.json
generated: '2026-08-13'
method: generated
source: openapi/adobe-suite-photoshop-openapi.json (23 operations, verified operationIds)
operations:
  - cutout
  - mask
  - icstatus
  - smartObject
  - renditionCreate
  - documentManifest
  - pitsstatus
---

# Automate a Photoshop edit

Base URL: `https://image.adobe.io`.

Auth is the same estate-wide pair — `Authorization: Bearer <IMS token>` plus
`x-api-key: <Client ID>`. See `authentication/adobe-suite-authentication.yml`.

## Inputs and outputs are your storage, not Adobe's

Unlike PDF Services, the Photoshop API does not host your files. Every request body names
an `input` href and an `output` href, both signed URLs into **your** storage (Adobe
documents Amazon S3, Azure Blob, Dropbox and Google Cloud Storage, or an Adobe Creative
Cloud asset path). Adobe reads from the first and writes to the second.

This matters for an agent: a successful job status does not mean you have the file. Go
read your own bucket.

## Remove a background

1. `cutout` — `POST /sensei/cutout` with `{input, output}`. Returns **202** and a job id.
2. `icstatus` — `GET /sensei/status/{jobId}` until terminal.
3. Fetch the result from your own `output` location.

`mask` (`POST /sensei/mask`) is the same shape and shares the `icstatus` poller.

## Work on a PSD

The PSD service is a separate path group with its own poller, `pitsstatus`
(`GET /pie/psdService/status/{jobId}`):

- `documentManifest` — read the layer tree before editing anything. Always do this first;
  layer indexes are not stable across documents.
- `smartObject` — replace a smart object's contents.
- `text` — edit text layers.
- `documentOperations` — apply a batch of PSD edits.
- `renditionCreate` — render the document to JPEG/PNG/TIFF/PSD at chosen sizes.
- `photoshopActions` / `actionJSON` / `actionJsonCreate` — run recorded Photoshop actions
  server-side.
- `productCrop`, `depthBlur`, `artboardCreate`, `documentCreate`.

## Lightroom-style edits live here too

`autoTone`, `autoStraighten`, `edit`, `presets` and `xmp` sit under `/lrService/` in this
same contract, polled with `acrstatus` (`GET /lrService/status/{jobId}`).

## Rules

- Three path groups, three status endpoints (`icstatus`, `pitsstatus`, `acrstatus`).
  Poll the one that matches the group you submitted to.
- No idempotency key. A retried POST re-renders and re-bills.
- `429` → back off with `Retry-After`. `410 Gone` is common here and means the job asset
  link has expired — resubmit rather than retry the fetch.
- adobe.io times out at 60 seconds; the job continues server-side, so poll rather than
  resubmit.

## Note on the newer contract

Adobe also publishes a Firefly Services Photoshop v2 API at
`https://photoshop-api.adobe.io` (`openapi/adobe-suite-firefly-photoshop-v2-openapi.json`,
7 operations). Check which one your entitlement covers before building against either.
