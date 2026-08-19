---
name: sensors-data-run-funnel-analysis
description: Run a funnel or retention analysis against a Sensors Analytics deployment and read the result, including drilling into the individual users behind a step.
api: Sensors Analytics OpenAPI (神策分析)
generated: '2026-08-13'
method: generated
source: openapi/sensors-data-analytics-model-v2-openapi.yml, openapi/sensors-data-analytics-event-meta-v1-openapi.yml, openapi/sensors-data-analytics-property-meta-v1-openapi.yml
operations:
  - ListEventsAll
  - ListAllEventProperties
  - GetPropertyValues
  - QueryFunnelReport
  - QueryFunnelUsers
  - QueryRetentionReport
  - QueryRetentionUsers
  - SqlQuery
---

# Run a funnel or retention analysis

Sensors Analytics is deployed per customer, so before anything else you need the cluster
base URL. Every call in this skill is `POST {base}/api/v3/analytics/v2{path}` with two
required headers:

```
api-key: <project-scoped key>
sensorsdata-project: <project identifier, e.g. default>
```

## 1. Discover what you are allowed to ask about

Never guess an event name. Fetch the real metadata first.

- `ListEventsAll` — `GET /event-meta/events/all` — every event defined in this project.
- `ListAllEventProperties` — `GET /property-meta/event-properties/all` — every event property.
- `GetPropertyValues` — `POST /property-meta/property/values` — candidate values for a
  property, so a filter you build actually matches rows.

## 2. Build the funnel query

`QueryFunnelReport` — `POST /model/funnel/report`. The request is composed from the shared
query vocabulary used across every analysis model in this contract: `EventStep`,
`FilterCondition` / `CompoundFilterCondition`, `TimeRange` or `TimeWindow`, and
`LookbackWindow` for the conversion window. Prefer interface version **v2**; v1 is still
published and still served, and both expose an operation called `QueryFunnelReport` with
different request shapes. Bind to the version in the path, not to the operationId.

## 3. Read the answer, then read the people

`QueryFunnelUsers` — `POST /model/funnel/users` — the individual users behind a step. This
is the operation that turns an analysis into an audience, and it is where personal data
enters your workflow. Treat the returned `user_id` / `first_id` / `second_id` as
identifiers, not as content to echo back to an end user.

The retention pair is symmetrical: `QueryRetentionReport` then `QueryRetentionUsers`.

## 4. Fall back to SQL only when the model cannot express it

`SqlQuery` — `POST /model/sql/query` — arbitrary SQL over the project's event and user
tables. It is the most powerful operation on the surface and the least constrained. Use it
when the funnel/retention/attribution models genuinely cannot express the question, and
never to work around a permission you were not granted — the api-key carries the
permissions of the account it was minted for, and SQL does not widen them.

## Rules that apply to every step

- **HTTP 200 is not success.** Parse the body and test `code == "SUCCESS"`. Failures come
  back inside the same envelope with a `message`. See `errors/sensors-data-problem-types.yml`.
- **Log `request_id`** from every response. It is the only handle support can trace.
- **Paginate with `page_index` / `page_size`**, maximum 100 per page; read `page.total`,
  `page.current_page` and `page.page_count` from the response.
- **There is no idempotency key.** Reads are safe to retry; do not blind-retry a write.
- **The spec is in Chinese.** Operation summaries and parameter descriptions are zh-CN.
