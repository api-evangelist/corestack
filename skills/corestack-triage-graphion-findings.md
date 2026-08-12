---
name: corestack-triage-graphion-findings
description: >-
  Triage application security findings in CoreStack Graphion — walk the portfolio hierarchy to the
  top actionable issues, pull SBOM vulnerability and container-finding detail, and check how widely
  a vulnerability is spread before deciding what to fix first.
api: CoreStack External API
base_url: https://api.corestack.io
spec: openapi/corestack-external-api-openapi-original.json
operations:
  - ListPortfolio
  - PortfolioHierarchy
  - TopActionableIssues
  - SummarySbomVulnerabilities
  - BatchSbomComponents
  - ListContainerFindings
  - OrganizationalVulnerabilityPrevalence
requires_skill: corestack-authenticate-and-scope
generated: '2026-08-11'
method: generated
source: openapi/corestack-external-api-openapi-original.json + https://docs.corestack.io/docs/mcp-tool-guide
---

# Triage Graphion application security findings

Graphion is CoreStack's CNAPP layer. Its data hangs off a four-level hierarchy —
**Portfolio → Application → Project → SBOM** — and every query is scoped by tenant. Run
`corestack-authenticate-and-scope` first.

## Steps

1. **Find the portfolio — `ListPortfolio`**
   `POST /v1/appsecops/portfolios/list`. Portfolios are the top of the Graphion hierarchy; pick the
   one covering the estate you are triaging.

2. **See the structure — `PortfolioHierarchy`**
   `POST /v1/appsecops/dashboard/summary/portfolio_hierarchy`. Returns portfolios with their
   applications and projects in one call. Do this before drilling — it saves three round trips and
   tells you whether the finding you are chasing belongs to a service anyone still owns.

3. **Start with what actually matters — `TopActionableIssues`**
   `POST /v1/appsecops/dashboard/top_actionable_issues`. Returns a risk-prioritised list spanning
   vulnerabilities, threats and configuration violations. This is the correct entry point: it is
   CoreStack's own prioritisation, and starting from a raw vulnerability list instead will bury you.

4. **Pull the SBOM vulnerability picture — `SummarySbomVulnerabilities`**
   `POST /v1/appsecops/dashboard/summary/sbom_vulnerabilities`. All SBOM vulnerabilities for the
   scope, so you can see whether the top issues cluster in one component.

5. **Identify the offending component — `BatchSbomComponents`**
   `POST /v1/appsecops/sbom_components/batch`. Batch-resolve component ids to full detail. Batch
   rather than looping — the related SBOM vulnerability batch endpoint documents a ceiling of
   **1,000 ids per request**, which is the only batch limit CoreStack publishes anywhere.

6. **Check the container surface — `ListContainerFindings`**
   `POST /v1/appsecops/container/findings/list`. Supports filtering and pagination. Container
   findings can also be *ingested* from external scanners (`IngestContainerFindings`) — that one is
   a write; do not call it during triage.

7. **Decide priority by blast radius — `OrganizationalVulnerabilityPrevalence`**
   `POST /v1/appsecops/vulnerabilities/prevalence/organization`. Tells you how widely a vulnerability
   is present across the organisation. A medium-severity CVE in forty services usually outranks a
   high-severity one in a single retired project, and this is the operation that lets you make that
   call with evidence.

## Conventions that matter here

- Graphion's newer operations use `service_account_id` where the older governance and cost paths use
  `cloud_account_id`. They refer to the same onboarded cloud account. See
  `data-model/corestack-data-model.yml`.
- All list operations here are POST with a filter body, not GET with a query string. They are reads.
- The 4xx/5xx body is always `{"message": "<string>"}` — no error code, no field pointer.

## What you cannot do from here

Graphion triage is read-only through this path. Remediation runs through the guardrail policy
surface (`ExecutePolicy`, `ExecuteRecommendation`), which applies changes to live cloud resources and
has **no idempotency mechanism** — a retried execution may run twice. Treat any remediation as a
separate, deliberately confirmed step, never as a continuation of triage.
