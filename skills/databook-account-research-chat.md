---
name: Research an account with Databook chat and deep reasoning
description: >-
  Ask Databook a question about a target account and carry the answer across turns, then
  escalate to a named reasoning agent when the question needs cited, structured analysis
  rather than a conversational reply.
api: openapi/databook-openapi-original.json
base_url: https://api.databook.com
operations:
  - chat_v1_chat_post
  - run_reasoning_v1_reasoning_post
  - list_insights_v1_insights_get
generated: '2026-08-13'
method: generated
source: openapi/databook-openapi-original.json
---

# Research an account with Databook chat and deep reasoning

Databook exposes two synchronous analytical surfaces. They are not interchangeable.

- `chat_v1_chat_post` — conversational. One question in, one message out, with a thread id you
  pass back to continue.
- `run_reasoning_v1_reasoning_post` — a named agent run. Returns a response plus **citations**
  and **metadata**, which is what you want when the answer has to be defensible.

## Headers on every call

```
Authorization: Bearer <YOUR_ACCESS_TOKEN>
Content-Type: application/json
databook-user-id: <user id>
databook-tenant-id: <tenant id>
```

The token is provisioned by Databook support. There is no self-serve signup and no OAuth flow.

## Conversational research

Call `chat_v1_chat_post` (`POST /v1/chat`) with `ChatRequest`:

- `query` (required, string) — the question.
- `conversation_id` (optional, string) — omit it on the first turn.

The `ChatResponse` returns `conversation_id` and `content`. **Capture `conversation_id` from the
first response and send it on every subsequent turn**, or each question is answered with no
memory of the last one. Databook does not expose an operation to list, fetch or delete
conversations — the id is yours to hold, and there is no server-side way to inspect or clean up
a thread.

## Deep reasoning

Call `run_reasoning_v1_reasoning_post` (`POST /v1/reasoning`) with `PostReasoningRequest`:

- `agent_name` (required, string) — the Databook agent to run.
- `parameters` (optional, object) — the agent's inputs.

`PostReasoningResponse` returns `response`, `reference_id`, `citations` and `metadata`. Surface
`citations` to the user whenever you present the answer — the citation set is the difference
between this endpoint and chat, and dropping it turns sourced analysis into an unattributed
claim. Keep `reference_id`; it is the correlation id Databook support asks for.

**Agent names are not discoverable through the API.** No operation lists the available
`agent_name` values and the OpenAPI does not enumerate them, so an agent name must come from
Databook or from the customer's own configuration. Do not guess one — an unknown name returns
`400 invalid_request`.

## Choosing the right surface

| Need | Use |
| --- | --- |
| A quick, iterative question about one account | `chat_v1_chat_post` |
| A structured, cited analysis from a named agent | `run_reasoning_v1_reasoning_post` |
| The same question across many accounts | the batch job flow — see `databook-batch-insight-run.md` |
| To know which analytical questions exist at all | `list_insights_v1_insights_get` |

## Timeouts and limits

Any single call is cut off at **180 seconds** and returns `503 server_side_overload_error`. Deep
reasoning is the operation most likely to hit it. Do not raise your client timeout above 180s
expecting a late answer, and do not fan out concurrent reasoning calls to work around it — the
API is rate limited (no number published) and will answer `429 rate_limit_error`. Move
multi-account work to the batch endpoints.

## Errors

All operations share one envelope: `{"error": {"type", "message"}, "reference_id"}`. See
`errors/databook-problem-types.yml` for the full status/type table. Neither POST is idempotent
and neither accepts an idempotency key, so a blind retry of `/v1/chat` or `/v1/reasoning` runs
the work again and bills it again.
