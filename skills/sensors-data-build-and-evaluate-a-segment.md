---
name: sensors-data-build-and-evaluate-a-segment
description: Define a customer segment in Sensors Horizon (神策数界), run its computation, wait for the task, and read the resulting membership — the core CDP flow.
api: Sensors Horizon CDP OpenAPI (神策数界)
generated: '2026-08-13'
method: generated
source: openapi/sensors-data-horizon-segment-v1-openapi.yml, openapi/sensors-data-horizon-schema-v1-openapi.yml
operations:
  - ListEventSchemas
  - ListSchemaFields
  - ListSegmentDefinitions
  - CreateSegmentDefinitionWithRule
  - UpdateSegmentDefinitionWithRule
  - EvaluateSegment
  - GetSegmentTask
  - CancelSegmentTask
  - ListSegmentItems
  - GetLastedSegmentItem
  - UpdateSegmentSchedulerStatus
---

# Build and evaluate a segment

Base: `{base}/api/v3/horizon/v1`. Headers `api-key` and `sensorsdata-project` are required
on every call. Note that almost every operation here is a **POST, including the reads** —
the contract has no path parameters, so filters travel in the body.

## 1. Learn the schema before writing a rule

- `ListEventSchemas` — `POST /schema/event/list`
- `ListSchemaFields` — `POST /schema/field/list`

A segment rule references fields by name. Reading the schema first is the difference
between a rule that computes and a rule that silently matches nobody.

## 2. Check whether the segment already exists

`ListSegmentDefinitions` — `POST /segment/definition/list`. There is no idempotency key on
this API, so creating twice creates two segments. Search before you create.

## 3. Create the definition together with its rule

`CreateSegmentDefinitionWithRule` — `POST /segment/definition/create-with-rule`. The rule
is a `SegmentRuleExpression`, which is a union of seven published shapes — pick the one
that matches the question:

| shape | use it for |
|---|---|
| `SimpleEventSequenceSegmentRuleExpression` | "did A then B" |
| `EventSequenceSegmentRuleExpression` | ordered sequences with windows |
| `GroupExpressionSegmentRuleExpression` | AND/OR groups of conditions |
| `CustomSegmentRuleExpression` | a custom predicate |
| `EqlSegmentRuleExpression` | an EQL expression |
| `SqlSegmentRuleExpression` | raw SQL |
| `LoadingSegmentRuleExpression` | an imported/loading population |

Use `UpdateSegmentDefinitionWithRule` to revise, `DeleteSegmentDefinition` /
`RecoverSegmentDefinition` for the soft-delete lifecycle.

## 4. Compute it — this is asynchronous

`EvaluateSegment` — `POST /segment/definition/evaluate` starts a computation and returns a
task. It does **not** return membership.

Then poll `GetSegmentTask` — `POST /segment/task/get`. If the caller abandons the request,
call `CancelSegmentTask` — `POST /segment/task/cancel` rather than leaving the job running;
segment computation is expensive on a customer's own cluster.

For a recurring segment, `UpdateSegmentSchedulerStatus` —
`POST /segment/definition/status/update` turns the schedule on and off.

## 5. Read the result

- `GetLastedSegmentItem` — `POST /segment/latest-item/get` — the most recent computed
  partition. (The operationId is misspelled in the published contract; send it exactly.)
- `ListSegmentItems` — `POST /segment/item/list` — historical partitions.

## 6. Hand it to marketing

A segment becomes reachable by creating an audience rule from it in Sensors Focus:
`CreateAudienceFilterRuleFromUserGroup` —
`POST /api/v3/focus/v1/express-audience-meta/rule/user-group-ref/create`. That call is the
seam between the CDP and campaign execution.

## Rules

- Test `code == "SUCCESS"`; HTTP status carries no outcome.
- Log `request_id`.
- No idempotency key: a retried create makes a second segment.
- Check `QueryQuota` on the Focus side before creating audience rules — projects have a
  documented resource quota, readable through the API.
