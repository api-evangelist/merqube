---
name: merqube-retrieve-index-levels
description: >-
  Pull an index's level history, portfolio composition over time, and performance statistics from
  the MerQube SecAPI and IndexAPI. Use this for backtesting inputs, tear sheets, and answering
  "what did this index return".
api: MerQube API
base_url: https://api.merqube.com
generated: '2026-08-25'
method: generated
source: >-
  Grounded in operations present in https://api.merqube.com/api (OpenAPI 3.1, info.version 4.40.0)
  and the "Get Index levels" example in
  https://merqube.stoplight.io/docs/api/p6s6m4p35ehfv-authentication
operations:
  - index_portfolio              # GET /index/{uuid}/portfolio
  - index_portfolio_allocations  # GET /index/{uuid}/portfolio_allocations
  - index_stats                  # GET /index/{uuid}/stats
  - index_caps                   # GET /index/{uuid}/caps
authentication: optional
mutating: false
---

# Retrieve MerQube index levels, portfolios and statistics

Creating an index in MerQube automatically creates two SecAPI securities that share the index's
name and namespace: an `index` security holding returns and metrics, and an `intraday_index`
security which carries metrics only for real-time indices. Levels come from the SecAPI; composition
and statistics come from the IndexAPI.

## 1. Levels — SecAPI, addressed by NAME

```
GET https://api.merqube.com/security/index?name={index_name}&metrics=price_return
Authorization: APIKEY {your_api_key}
```

This is the call MerQube's own documentation gives for "Get Index levels". Use
`security/intraday_index` for the real-time series. Discover what is available first with:

```
GET https://api.merqube.com/security/index/metrics
```

Note the asymmetry: the SecAPI reads addressed **by name and namespace**, while the IndexAPI reads
address **by UUID**.

## 2. Portfolio history — IndexAPI, addressed by UUID

```
GET https://api.merqube.com/index/{uuid}/portfolio?start_date=2024-01-01&end_date=2024-12-31
GET https://api.merqube.com/index/{uuid}/portfolio_allocations
```

`start_date` and `end_date` accept the `MerqTimestamp` shape — `2021-01-01`,
`2021-01-01T01:01:01`, or with microseconds.

Positions are **polymorphic with no discriminator**. Branch on which identifier field is present:
`ric` (Refinitiv), `fsym_id` (FactSet), a SecAPI id, a MerQube index reference, or option/futures
fields. Do not assume one shape.

## 3. Statistics and caps

```
GET https://api.merqube.com/index/{uuid}/stats
GET https://api.merqube.com/index/{uuid}/caps
```

`/stats2` exists alongside `/stats` and shares the `index_stats` operationId. The contract does not
say how they differ — prefer `/stats` and treat `/stats2` as unspecified.

## 4. Check whether the number is fresh

```
GET https://api.merqube.com/index/{uuid}/run_state
```

Returns `PENDING_CREATION`, `RUNNING`, `SUCCEEDED` or `FAILED`, plus an `error` string on failure.
MerQube publishes no status page, so **this is the only health signal available**. Check it before
reporting a level as current.

## Consistency

If you have just written data and need to read it back, leave `prefer_read_replica` at its default
`false`. Setting it to `true` opts into replica and SecAPI cache reads, which are faster but may not
reflect your write.

## Errors

| Status | Meaning |
| --- | --- |
| 400 | Bad metric requested, bad portfolio or request, or an illegal date value. |
| 403 | Not authorized to see this namespace. |
| 404 | Index, portfolio or security does not exist. |
