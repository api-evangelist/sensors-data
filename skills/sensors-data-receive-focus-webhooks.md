---
name: sensors-data-receive-focus-webhooks
description: Stand up an HTTP endpoint that receives Sensors Focus webhook callbacks, answer them correctly, and close the loop with delivery receipts.
api: Sensors Focus OpenAPI (神策智能运营)
generated: '2026-08-13'
method: generated
source: asyncapi/sensors-data-webhooks.yml, https://manual.sensorsdata.cn/sf/docs/webhook_integration
operations:
  - QueryChannel
  - QueryChannelInstance
  - CreateChannelInstance
  - UpdateChannelInstance
  - ExecuteTestSend
  - CheckChannelInstanceContent
  - GetChannelInstanceMetricsDefine
---

# Receive Sensors Focus webhooks

This is the one Sensors Data surface where **the platform calls you**. Sensors Focus posts
to an HTTP endpoint you own whenever a user satisfies a marketing plan's conditions, and
your endpoint decides what actually happens — SMS, push, coupon issuance, an in-app
message, a write to your own systems.

## 1. The body is a LIST, not an object

Focus micro-batches multiple triggered users into a single request. The first thing that
breaks a naive receiver is deserializing the body as one object.

```
POST /your/path HTTP/1.1
Content-Type: application/json;charset=UTF-8

[ { "project_name": "...", "sf_version": "...", "user_profile": {...},
    "receipt_properties": {...}, "plan_info": {...}, "params": {...},
    "send_id": "..." }, ... ]
```

Field names are `snake_case`. The docs explicitly warn camelCase frameworks to configure
deserialization mapping.

## 2. Every params value is a string

Whatever type an operator configured — integer, decimal, date, percentage — arrives as a
string. A percentage arrives as its numeric value with no `%`. Coerce on your side; do not
assume the JSON type.

## 3. Answer in exactly one of three ways

| response | meaning |
|---|---|
| `200`, empty body | the whole batch succeeded |
| `200` + `[{"succeed": true}, {"succeed": false, "fail_reason": "..."}]` | per-entry result, **index-aligned with the request list** |
| any non-200 | the whole batch failed |

There is no partial-success status code. If one entry in fifty fails and you return a 500,
you have just told Focus that all fifty failed.

## 4. Verify the signature

Configure a Secret Token on the Focus channel and verify it on every request. The
first-party Java helper (`github.com/sensorsdata/sf-webhook-helper`) enforces the check
whenever `secretTokenForSignatureCheck` is supplied to `Bootstrap`. If you supply no token,
your endpoint is an unauthenticated write path into your own messaging systems.

## 5. Close the loop with a receipt

Delivery reporting inside Focus is blind unless you report back. Send a normal Sensors
Analytics event through any server SDK:

- event name `$PlanMsgArrived`
- `$sf_msg_status` = `RECEIPT_SEND_SUCCESS` or `RECEIPT_SEND_FAILED` (upper case, required)
- `$sf_channel_category` = `WEBHOOK` (upper case, required)
- echo the rest of `receipt_properties` back verbatim — `$sf_plan_id`, `$sf_plan_version`,
  `$sf_plan_strategy_id`, `$sf_strategy_unit_id`, `$sf_channel_id`, `$sf_component_id`,
  `$sf_enter_plan_time`, `$sf_send_time` — plus `$sf_send_fail_code` / `$sf_fail_reason`
  on failure.

## 6. Manage the channel through the API

- `QueryChannel` / `QueryChannelInstance` — `GET /express-action-channel/channel/query`,
  `/channel/instance/query`
- `CreateChannelInstance` / `UpdateChannelInstance` — `POST .../channel/instance/create`,
  `/update`
- `ExecuteTestSend` — `POST .../channel/instance/test-send/execute` — the closest thing to
  a sandbox on this platform: exercise a channel against a single target before a plan
  goes live.
- `CheckChannelInstanceContent` — `POST .../channel/instance/send-content/check` —
  validate message content before delivery.

## Timing expectations

Webhook delivery "makes no timeliness guarantee". Micro-batching adds several seconds. If
you need sub-second reach, the docs direct you to the in-app popup product instead, not to
a tuning parameter.
