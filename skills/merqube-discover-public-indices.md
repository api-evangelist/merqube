---
name: merqube-discover-public-indices
description: >-
  Find MerQube indices and read their manifests without any credential, using the world-readable
  `default` namespace. Use this to answer "does MerQube publish an index for X" and to resolve an
  index name to the permanent UUID every other call needs.
api: MerQube API
base_url: https://api.merqube.com
generated: '2026-08-25'
method: generated
source: >-
  Grounded in operations present in https://api.merqube.com/api (OpenAPI 3.1, info.version 4.40.0)
  and the authorization model documented at
  https://merqube.stoplight.io/docs/api/p6s6m4p35ehfv-authentication
operations:
  - index                 # GET /index
  - index_uuid            # GET /index/{uuid}
authentication: optional
mutating: false
---

# Discover MerQube indices

MerQube serves a genuinely public read surface. Everything in the `default` namespace — the
MerQube-branded index families — is readable with **no API key at all**. Only customer namespaces
require one.

## 1. Search by keyword

```
GET https://api.merqube.com/index?filter=MQU&fields=name,title,description&page=1&page_size=100
```

- `filter` is a substring match over `name` and `title`.
- `fields` is a sparse fieldset; `id` and `namespace` are always returned regardless.
- **Always send `fields` and `page_size`.** A bare `GET /index` returns the entire collection —
  measured at 11.8 MB on 2026-08-25 — because there is no default page cap on the response.

## 2. Or look up an exact name

```
GET https://api.merqube.com/index?names=MQUSTB20,MQUSTRAV
```

`name` takes one exact name, `names` a comma-delimited list. `ids` takes UUIDs and is ignored when
`names` is also present.

## 3. Read the result envelope, not just `results`

Every response is `{results, error_codes, deprecation_warnings, linked_resources}`.

**Check `error_codes` even on an HTTP 200.** Code `00001` / `RESULTS_WERE_FILTERED` means rows
matching your query were withheld because your credential does not cover their namespace. An
unauthenticated caller will see this routinely. Silence about it is how a partial answer gets
reported as a complete one.

## 4. Keep the UUID, not the name

```
GET https://api.merqube.com/index/{uuid}
```

Names are mutable; `id` is permanent. MerQube's own docs state the only way to change an id is to
delete the object and recreate it. Persist the UUID and treat any stored name as a cache.

## Errors

| Status | Meaning |
| --- | --- |
| 403 | Not authorized to see this index namespace. MerQube returns **403, not 401**, when a key is missing. |
| 404 | Index does not exist — usually a stale name lookup after a rename. |

## Notes

- No rate limits are published and no `RateLimit-*` or `Retry-After` header is returned. Pace
  yourself; there is no runtime signal to back off on.
- Add `format=csv` for a CSV body instead of JSON.
- Every response carries an `x-request-id` header — quote it to support@merqube.com.
