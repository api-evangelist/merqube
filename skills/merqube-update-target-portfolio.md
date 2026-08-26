---
name: merqube-update-target-portfolio
description: >-
  Upload or correct the constituents a MerQube index will rebalance into on a future date, using
  the effective-timestamp replace semantics and the status compare-and-set token. Use this for the
  recurring "push tomorrow's basket" flow.
api: MerQube API
base_url: https://api.merqube.com
generated: '2026-08-25'
method: generated
source: >-
  Grounded in paths present in https://api.merqube.com/api (OpenAPI 3.1, info.version 4.40.0). The
  timing rule is quoted from MerQube's TargetPortfolio documentation at
  https://merqube.stoplight.io/docs/api/p6s6m4p35ehfv-authentication
operations:
  - GET /index/{uuid}/target_portfolio
  - PUT /index/{uuid}/target_portfolio
  - GET /index/{uuid}/target_portfolio/{target_portfolio_uuid}
  - PATCH /index/{uuid}/target_portfolio/{target_portfolio_uuid}
  - DELETE /index/{uuid}/target_portfolio/{target_portfolio_uuid}
authentication: required
mutating: true
---

# Update a MerQube index target portfolio

A TargetPortfolio is the set of constituents an index rebalances into on a stated future effective
date. This is the highest-frequency write on the MerQube API.

## The timing rule — read this first

MerQube's documentation states it plainly:

> Make sure to create a TargetPortfolio for date T **before** the index has computed date T.
> TargetPortfolios with dates in the past will be ignored.

This is the one write on this API with a stated boundary, and it is a **computation cutoff, not a
retention window**. Once the index has run for T, an upload for T is silently ignored — no error,
no rejection. Verify the run has not happened yet:

```
GET https://api.merqube.com/index/{uuid}/run_state
```

`SUCCEEDED` for the date you are targeting means you are too late.

## 1. Read what is there now

```
GET https://api.merqube.com/index/{uuid}/target_portfolio
Authorization: APIKEY {your_api_key}
```

Keep the returned `status` block for the object you intend to modify.

## 2. Upload — PUT is an upsert keyed on `eff_ts`

```
PUT https://api.merqube.com/index/{uuid}/target_portfolio
Authorization: APIKEY {your_api_key}
Content-Type: application/json
```

Each uploaded portfolio is evaluated on its own: if no portfolio exists with that `eff_ts` one is
created; if one does, **the upload replaces it**. Re-PUT with the same `eff_ts` is therefore the
normal correction path — you do not need to delete first.

Positions are polymorphic. Choose the shape that matches your identifier scheme — RIC, FactSet
`fsym_id`, a SecAPI id, a MerQube index reference, an option, or a future — and be consistent
within one portfolio.

## 3. Correct or remove a single portfolio

```
PATCH  https://api.merqube.com/index/{uuid}/target_portfolio/{target_portfolio_uuid}
DELETE https://api.merqube.com/index/{uuid}/target_portfolio/{target_portfolio_uuid}
```

`PATCH` updates only the fields you send.

## Concurrency — there is no Idempotency-Key

MerQube ships **optimistic concurrency instead of idempotency**. Every object carries a `status`
block (`created_at`, `created_by`, `last_modified`, `last_modified_by`). It must be supplied on
`PUT` and `PATCH`, and the write is only accepted if it matches the copy in storage.

Consequences for an agent:

- A stale `status` block means your write is rejected, not silently merged. Re-read, re-apply,
  re-send.
- A retried request whose response you never saw is **not** automatically safe. Re-read the object
  and compare before retrying.
- `status.locked_after`, if set, locks the manifest against all other edits after that timestamp. To
  edit, first `PUT`/`PATCH` that field to `null` or to at most one hour in the future, then make
  the change.

## Errors

| Status | Meaning |
| --- | --- |
| 400 | Malformed body, or an `eff_ts` that is not a valid date-time. |
| 403 | Key does not cover the index's namespace. |
| 404 | Index or target portfolio with those UUIDs does not exist. |

Check `error_codes` in the response envelope even on a 200.

## Automation

The first-party Python client installs an `update_portfolio` console script for the equity-basket
case (`pip install merqube-client-lib`).
