---
name: corestack-investigate-cost-anomaly
description: >-
  Investigate an unexpected cloud spend spike in CoreStack FinOps — find the anomaly, drill to the
  resources causing it, and check it against the budget it broke.
api: CoreStack External API
base_url: https://api.corestack.io
spec: openapi/corestack-external-api-openapi-original.json
operations:
  - CostAnomalySummary
  - CostAnomalyResources
  - GetCostAggregation
  - GetCostAggregationTrend
  - ListDimensions
  - ListBudget
  - ViewBudget
requires_skill: corestack-authenticate-and-scope
generated: '2026-08-11'
method: generated
source: openapi/corestack-external-api-openapi-original.json + https://docs.corestack.io/docs/mcp-tool-guide
---

# Investigate a cost anomaly

Run `corestack-authenticate-and-scope` first. You need a valid token, `X-Auth-User`, and the
`tenant_id` for the department whose spend you are investigating.

## Steps

1. **Find the anomalies — `CostAnomalySummary`**
   `POST /cost_anomaly/billing_cost_anomaly`. Returns account-wise daily billing cost anomalies for
   your scope. This is the same data the console Anomaly Summary page renders. Start here rather
   than at aggregation: the anomaly detector has already done the comparison against expected spend.

2. **Drill to the resources — `CostAnomalyResources`**
   `POST /cost_anomaly/billing_cost_anomaly_resources`. Takes the anomaly you selected and returns
   the resource-level detail underneath it. This is where the actual answer usually is — a resized
   instance family, a forgotten data-transfer path, a new region.

3. **Confirm the shape of the spend — `GetCostAggregationTrend`**
   `POST /v2/billing/aggregation/trend`. Returns a **time series**, so use it to see whether the
   anomaly is a one-day spike or the start of a new run rate. Set `granularity` to `86400` for daily
   points; the default is `2592000` (monthly), which will hide a one-day spike entirely.
   If you want a single total rather than a series, use `GetCostAggregation`
   (`POST /v2/billing/aggregation`) instead — same filters, no time axis.

4. **Group it properly — `ListDimensions`**
   `POST /v1/dimensions/list`. CoreStack groups cost by *dimensions*, and the useful ones are
   tenant-specific (business unit, environment, cost centre, application). Fetch the available
   dimensions before choosing a `group_by`, rather than guessing at provider-native fields.

5. **Check what it broke — `ListBudget` then `ViewBudget`**
   `POST /budget/dashboard/list_budgets` returns budgets with health indicators; filter to the ones
   covering the affected scope. Then `GET /budget/{budget_id}/view` for the detail of the specific
   budget that moved into alert.

## Conventions that matter here

- **POST does not mean write.** Every operation in this skill is a read; CoreStack uses POST for
  queries whose filters are too complex for a query string. Retrying any of them is safe.
- **Pagination** on LIST actions is `?limit=25&page=2`. The paged response envelope is not
  documented, so count returned records against your `limit` to decide whether to ask for another
  page rather than looking for a next-page token.
- **Dates** are ISO-8601 with time, in UTC.
- **No rate limits are published** (`rate-limits/corestack-rate-limits.yml`) and no `Retry-After`
  header exists. Pace your own requests; you will get no runtime signal that you are going too fast.

## Failure handling

- `400 {"message": ...}` — the filter tree or date range is malformed. The message is free text with
  no field pointer; bisect your filter rather than parsing the string.
- `401` — token expired mid-run. Refresh (max three times) and resume; all steps here are safe to
  repeat.
- `500` — do **not** blind-retry a write. Every operation in this skill is a read, so retry is fine
  here, but see `conventions/corestack-conventions.yml`: the API publishes no idempotency mechanism,
  so the same reasoning does not carry to the policy and budget write paths.
