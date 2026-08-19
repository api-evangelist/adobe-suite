---
name: Generate and retrieve an image with the Adobe Firefly API
description: >-
  Mint an IMS token with the Firefly scopes, submit an asynchronous image generation job,
  poll it to completion, and handle the trial-exhaustion and rate-limit signals correctly.
api: openapi/adobe-suite-firefly-openapi.json
generated: '2026-08-13'
method: generated
source: openapi/adobe-suite-firefly-openapi.json (14 operations, verified operationIds)
operations:
  - generateImagesV3Async
  - firefly_image_v5_generate_async_v4
  - jobResultV3
  - cancelJobV4
  - storageImageV2
  - getCustomModels
---

# Generate an image with Firefly

Base URL: `https://firefly-api.adobe.io`.

## 1. Mint a token with the right scopes

Firefly will not accept a bare Adobe token. `POST` to
`https://ims-na1.adobelogin.com/ims/token/v3` with `grant_type=client_credentials` and the
scope list Adobe publishes verbatim:

```
openid,AdobeID,session,additional_info,read_organizations,firefly_api,ff_apis
```

Send `Authorization: Bearer <token>` and `x-api-key: <Client ID>` on every call.
The token lasts 24 hours. See `scopes/adobe-suite-scopes.yml`.

## 2. Choose the model generation

- `generateImagesV3Async` — `POST /v3/images/generate-async`, the Image 3 family.
- `firefly_image_v5_generate_async_v4` — `POST /v4/images/generate-async`, Image 5.

Both are asynchronous and return a job handle, never an image.

If your prompt references an uploaded reference image, upload it first with
`storageImageV2` (`POST /v2/storage/image`) and pass the returned upload ID. Only these
storage domains are accepted for externally hosted images: `amazonaws.com`,
`windows.net`, `dropboxusercontent.com`, `storage.googleapis.com`,
`frontdoor.prod.azure.cxp.adobe.com`.

To use a tuned model, list what you have with `getCustomModels` (`GET /v3/custom-models`)
first.

## 3. Poll the job

`jobResultV3` (`GET /v3/status/{jobId}`) until terminal. To abandon a job, call
`cancelJobV4` (`PUT /v3/cancel/{jobId}`) — Adobe documents cancellation as idempotent, so
repeat cancels are safe.

## 4. Respect the two limits that actually bind

Firefly's published default is **4 requests per minute per organization** and **9,000
requests per day**. The daily ceiling stays at 9,000 even when the per-minute rate is
raised by an account manager, unless renegotiated separately. Serialize generation work;
do not fan out.

- `429 Too Many Requests` → back off, honour `Retry-After`, retry.
- `402 Payment Required` with "Trial Limit Exceeded" → **stop**. This is quota
  exhaustion, not throttling. Retrying will never succeed and costs nothing but noise.

## 5. Do not blind-retry a generation POST

There is no idempotency key on any Firefly operation. A retried generate is a second
billed generation with a different result. If a POST times out (adobe.io times out at 60
seconds), poll for the job rather than resubmitting.

## Sibling contracts

The rest of Firefly Services follows the same token, the same async shape and the same
limits: `openapi/adobe-suite-firefly-photoshop-v2-openapi.json`,
`adobe-suite-firefly-lightroom-openapi.json`,
`adobe-suite-firefly-illustrator-openapi.json`,
`adobe-suite-firefly-indesign-openapi.json`,
`adobe-suite-firefly-express-openapi.json`,
`adobe-suite-firefly-substance-3d-openapi.yaml`,
`adobe-suite-firefly-audio-video-openapi.json`,
`adobe-suite-firefly-translate-lipsync-openapi.json`,
`adobe-suite-firefly-workflow-builder-openapi.yaml`.
