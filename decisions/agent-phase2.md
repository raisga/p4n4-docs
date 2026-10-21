# Incident Investigator: Phase 2 (actions)

Status: design note. Phase 1 ([reference](../reference/agent.md)) only reads. This note
describes how the agent could take actions (write to Letta, send n8n alerts, publish
actuator commands) without doing any of them twice or without consent.

## The problem

A durable agent reruns work after a crash. For a read, a rerun only costs time. For an
action, it can repeat the effect: a second alert to the on-call engineer, a fault logged
twice in a machine's history, a valve opened again after someone closed it. The process
can die after the effect but before its result is committed, and the agent then cannot
know whether the effect happened.

## Unsafe replay by default

Every action tool keeps Pi Durable's default replay policy, `unsafe`. When a crash
interrupts such a call, it is never run again: the model gets an `interrupted` error
result with whatever output was committed, and decides what to do next with the other
tools. A replay-safe tool must not call an action tool as a nested call either:
replaying the caller would reattach to the nested call, but an unfinished one follows its
own `unsafe` policy and is abandoned.

## Operation IDs

Every effect carries an operation ID, so the receiver can drop a repeat:

```
<incident>:<tool>:<hash>    pump-2:2026-10-19T10:00:00.000Z:notify:3f9a1c
```

`hash` is a short digest of the tool's canonical arguments. The ID is derived, not
stored: when the model calls the tool again with the same arguments after an
`interrupted` result, the retry carries the same ID and the receiver drops it if the
first attempt went through. (`api.memo()` can't carry an ID across calls: its memos
belong to one tool task, and the retry is a new one.) An action that may legitimately
repeat with the same arguments, such as a second reminder, adds an explicit `attempt`
argument.

| Action | Where the ID goes | How the receiver deduplicates |
|--------|-------------------|-------------------------------|
| Letta: record a fault | the archival memory text, `[op:<id>]` | search for the ID before inserting |
| n8n: send an alert | `X-Operation-Id` header on the webhook | an n8n workflow step that remembers IDs (static data or a table) for a day |
| Actuator command | `commands/<device>/<action>` payload `{"op": "<id>", ...}` | the device or its Node-RED flow ignores an ID it has applied |

## Human approval

Actuator commands, and anything a site marks as needing one, go through approval:

1. A `beforeTool` hook on `ToolTask` blocks the call and writes a pending request into an
   approval document of the incident: tool, arguments, operation ID, and the reason the
   model gave.
2. The request is published to `agent/approvals/<device>` and listed by the status API,
   so p4n4-api and the dashboard can show it.
3. An operator approves or rejects it through the API, which commits the decision to the
   document. The model gets the outcome as the tool's result: "approved and sent", or
   "rejected by <user>: <note>".
4. Requests expire after a deadline the site sets; an expired request reads as rejected.

The decision is committed state, so a restart in the middle neither loses a pending
request nor repeats an approved one.

## Still open

- Which actions may skip approval (an n8n alert probably can, a valve can't), and who
  decides: per site, in `.env` or a config file.
- Whether approvals belong in p4n4-api's role model (operators approve, viewers see).
- How long to keep operation IDs on the receiving side.
