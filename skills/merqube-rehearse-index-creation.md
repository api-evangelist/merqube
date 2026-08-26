---
name: merqube-rehearse-index-creation
description: >-
  Render a complete MerQube index manifest from a strategy template WITHOUT creating anything, then
  review it before any write. Use this whenever the task is "design an index like X" — it is the
  only rehearsal surface this API has.
api: MerQube API
base_url: https://api.merqube.com
generated: '2026-08-25'
method: generated
source: >-
  Grounded in operations present in https://api.merqube.com/api (OpenAPI 3.1, info.version 4.40.0).
  The safety posture is derived from conventions/merqube-conventions.yml (no dry-run mode, no
  published reversal window).
operations:
  - helper_template_decrement          # POST /helper/index-template/decrement
  - helper_template_equity_index       # POST /helper/index-template/equity_index
  - helper_template_single_option      # POST /helper/index-template/single_option
  - helper_template_multieb            # POST /helper/index-template/multi_eb
  - helper_template_buffer             # POST /helper/index-template/buffer_simple
  - helper_template_defined_outcome    # POST /helper/index-template/defined_outcome
  - helper_template_vol_target         # POST /helper/index-template/vol_target
  - helper_template_sstr               # POST /helper/index-template/sstr
  - helper_solver_defined_outcome      # POST /helper/solver/defined_outcome
authentication: required
mutating: false
---

# Rehearse a MerQube index before creating it

The `/helper/index-template/*` family performs **server-side templating**: you post the strategy
parameters and MerQube returns a fully-formed index manifest. It does not create an index. This is
the closest thing the MerQube API has to a dry-run, and it is the correct first step for any index
design task.

## 1. Get an example payload

The first-party Python client ships a `get_example` console script for exactly this:

```
pip install merqube-client-lib
get_example
```

## 2. Render the manifest

Pick the template that matches the strategy family:

| Strategy | Endpoint |
| --- | --- |
| Decrement | `POST /helper/index-template/decrement` |
| Equity index / static basket | `POST /helper/index-template/equity_index`, `/static_basket` |
| Single option overlay | `POST /helper/index-template/single_option` |
| Multi equity basket | `POST /helper/index-template/multi_eb`, `/multi_eb_portfolios` |
| Buffer | `POST /helper/index-template/buffer_simple` |
| Defined outcome | `POST /helper/index-template/defined_outcome` |
| Vol target | `POST /helper/index-template/vol_target` |
| SSTR | `POST /helper/index-template/sstr` |
| Option strategies | `POST /helper/index-template/option_strategies` |

```
POST https://api.merqube.com/helper/index-template/decrement
Authorization: APIKEY {your_api_key}
Content-Type: application/json
```

A `400` here means the template parameters are wrong — fix them at this stage, where nothing has
been created.

## 3. Solve, if the target is an outcome rather than a parameter set

```
POST https://api.merqube.com/helper/solver/defined_outcome
POST https://api.merqube.com/findstrike     # find_strike_for_given_budget
POST https://api.merqube.com/optionprice    # get_option_price
```

## 4. Review the rendered manifest before writing

Check at minimum: `name`, `namespace`, `stage`, `launch_date`, `base_date`, `currency`,
`calc_freq`, `rebal_freq`, `calculation_check`, `benchmark`, and — most consequentially —
`identifiers[]`, which is what causes the index to disseminate to Bloomberg, Reuters, FactSet,
Nasdaq, Morningstar or Wind.

## STOP HERE unless a human has approved the write

`POST /index` creates a production index configuration. MerQube publishes:

- **no dry-run mode** on the write itself,
- **no retention or undelete window** for `DELETE /index/{uuid}`,
- **no statement** of what happens to the two SecAPI securities an index creates when it is deleted,
- **no manifest version history**, so reverting a `PUT` requires that you kept the prior copy.

Deletion also destroys the UUID permanently — MerQube's docs are explicit that recreating an object
with the same name produces a different id. Treat every write on this API as one-way and get
explicit human sign-off first.

## Reference

- Reversal paths and what is known about their windows: `conventions/merqube-conventions.yml`
- Manifest field semantics: `data-model/merqube-data-model.yml`
