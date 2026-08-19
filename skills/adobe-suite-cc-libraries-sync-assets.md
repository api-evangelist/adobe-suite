---
name: Read and update Creative Cloud Libraries assets
description: >-
  List a user's Creative Cloud Libraries, walk the elements inside one, upload a new asset,
  and archive or restore elements — the read/write path for brand assets, colors and
  character styles.
api: openapi/adobe-suite-cc-libraries-openapi.json
generated: '2026-08-13'
method: generated
source: openapi/adobe-suite-cc-libraries-openapi.json (27 operations, verified operationIds)
operations:
  - getLibraries
  - getLibrary
  - getLibraryElements
  - postLibraryComponent
  - postLibraryElements
  - archiveLibraryElements
  - unArchiveLibraryElements
  - search
---

# Work with Creative Cloud Libraries

Base URL: `https://cc-libraries.adobe.io`.

Auth is the estate pair — `Authorization: Bearer <IMS token>` plus `x-api-key`. This API
acts **on behalf of a user**, so the token must come from a user-authentication flow
(OAuth Web / SPA / Native App), not a Server-to-Server credential.

## 1. Enumerate

`getLibraries` (`GET /api/v1/libraries`) returns the libraries the authenticated user can
see. `getLibrary` (`GET /api/v1/libraries/{libraryId}`) reads one.

`search` (`POST /api/v1/search`) queries across libraries and elements — prefer it over
listing and filtering client-side.

## 2. Walk the contents

`getLibraryElements` (`GET /api/v1/libraries/{libraryId}/elements`) lists the elements —
colors, character styles, graphics, brushes, layer styles. `getLibraryElement` reads one.

Public libraries have unauthenticated read siblings: `getPublicLibrary`,
`getPublicLibraryElements`, `getPublicLibraryElement` under `/api/v1/public/libraries/`.
Use those when you only need to read a shared library and do not want to hold a user token.

## 3. Write

- `postLibraryComponent` (`POST /api/v1/libraries/{libraryId}/representations/content`) —
  upload an asset's bytes.
- `postLibraryElements` (`POST /api/v1/libraries/{libraryId}/elements`) — create, copy or
  move elements.
- `updateLibraryElements` (`PUT .../elements/metadata`) — update element metadata.
- `putLibraryElementRepresentations` — add, update or delete the renditions attached to an
  element.
- `patchLibrary` (`PATCH /api/v1/libraries/{libraryId}`) — applies multiple operations to
  one library **serially**; use it instead of racing several single-element writes.

## 4. Archive, do not delete

Creative Cloud Libraries has a two-stage delete and an agent must not confuse them:

- `archiveLibraryElements` / `archiveLibraryElement` (`DELETE .../elements`) — moves to the
  archive. Recoverable.
- `unArchiveLibraryElements` (`POST .../archive`) — restores.
- `deleteLibraryElements` / `deleteLibraryElement` (`DELETE .../archive/...`) —
  **permanent**. Do not call these without explicit human confirmation.
- `deleteLibrary` (`DELETE /api/v1/libraries/{libraryId}`) — permanent, removes everything.

## 5. Stay in the budget

Adobe publishes 20 requests per second per client ID for the Cloud Storage and
Collaboration surface, with `Retry-After` on 429. Bookmarks (`getBookmarks`,
`addLibraryBookmarks`, `putLibraryBookmark`, `removeLibraryBookmark`) share that budget.

## Events instead of polling

Creative Cloud Libraries publishes webhooks through Adobe I/O Events, so a sync integration
should subscribe rather than poll. See `asyncapi/adobe-suite-webhooks.yml`.
