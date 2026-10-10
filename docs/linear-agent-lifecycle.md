# Linear Agent Lifecycle Reference

This document captures the Linear agent lifecycle details the router needs to support when connecting Linear Agent Sessions to Orca-managed workers.

## Purpose

The Linear → Orca router should translate Linear agent-session events into durable router jobs and Orca worker actions, then translate Orca worker lifecycle updates back into Linear Agent Activities.

At a high level:

```text
Linear AgentSessionEvent
  → webhook receiver
  → durable router job/session
  → Orca worktree / supervised worker
  → Linear Agent Activities
```

Linear content, including issue descriptions, comments, prompt context, activity bodies, and attachments, must be treated as untrusted user input.

## Linear Agent Session states

Linear exposes six visible `AgentSession` states:

| Linear state | Meaning | Router implication |
| --- | --- | --- |
| `pending` | Session exists, but the agent has not acknowledged or started meaningful work yet. | Avoid staying here. On `created`, emit an activity or set `externalUrls` within Linear's expected window. |
| `active` | Agent is working. | Worker is queued, starting, or running. Emit concise progress via `thought` or `action`. |
| `awaitingInput` | Agent is waiting for a user response. | Router should have a pending question/decision and should resume on a later `prompted` webhook. |
| `complete` | Agent has finished successfully. | Emit a final `response` activity and mark the router job/session complete. |
| `error` | Agent failed or cannot continue. | Emit an `error` activity and preserve internal traces for debugging. |
| `stale` | Session became unresponsive or stale. | Prevent with prompt acknowledgement, durable recovery, and periodic status for queued/offline work. |

The router should not rely on manually setting these states. Linear derives session lifecycle from emitted Agent Activities and session updates.

## Incoming webhook actions

Linear sends `AgentSessionEvent` webhooks to the configured agent app. The router must acknowledge webhook HTTP requests within five seconds after durable acceptance.

### `created`

A new Agent Session was created, usually because a user mentioned the agent or delegated an issue to the agent.

Required behavior:

1. Validate webhook signature and freshness.
2. Deduplicate the delivery.
3. Persist the event and acknowledge the webhook quickly.
4. Within roughly ten seconds of receiving the created event, either:
   - emit an Agent Activity, typically `thought`, or
   - update the session `externalUrls`.
5. Create or resume the router job.
6. Route to an authorized Orca execution target.
7. Start an Orca worker or leave the job explicitly queued if no host is available.

Suggested mapping:

```text
AgentSessionEvent.created
  → router session accepted
  → emit thought: "I've started / queued this task"
  → route workspace/team/project/issue to Orca project/repo/host/agent
  → create Orca worktree or supervised worker
```

Use the webhook `promptContext` and structured issue/session fields to build the initial agent prompt, but treat all of it as untrusted task data.

### `prompted`

A user sent a message into an existing Agent Session. This can be a follow-up, an answer to an elicitation, or a control signal such as stop/cancel.

Required behavior:

1. Validate, deduplicate, persist, and acknowledge quickly.
2. Look up the existing session mapping.
3. Deliver the prompt to the existing Orca worker/conversation.
4. Preserve ordering per Linear agent session.
5. If no active mapping exists, reconcile before deciding whether to restart, create a new worker, or emit an error.

Suggested mapping:

```text
AgentSessionEvent.prompted
  → find router session by Linear agentSessionId
  → append prompt to per-session input queue
  → deliver to existing Orca dispatch/terminal
  → if answering a pending elicitation, resume the worker
```

The router should not start a fresh worker for every `prompted` event by default. Follow-ups belong to the existing Linear session and should route to the existing Orca worktree/worker unless policy says otherwise.

## Outgoing Agent Activity types

The router/agent communicates back to Linear by creating Agent Activities.

### `thought`

A progress/status update. Use for acknowledgement and lightweight ongoing progress.

Examples:

- `I've accepted this task and am preparing an Orca worktree.`
- `The task is queued because execution host dev-worker-1 is offline.`
- `I am inspecting the repository and issue context.`

Use `thought` for the first quick acknowledgement after `created`.

### `action`

A concrete operation or tool invocation. May include a result when completed.

Examples:

- creating an Orca worktree,
- launching an agent,
- running tests,
- opening a pull request,
- attaching a result link.

Use `action` for major externally visible operations, not for every low-level command.

### `elicitation`

A request for clarification, approval, authentication, or a user decision. This should move the session into an awaiting-input posture.

Examples:

- `Which repository should I use?`
- `Do you approve running the migration?`
- `Please choose one of these candidate repos.`

Router behavior:

```text
Orca worker asks question
  → emit Linear elicitation
  → store pending question/decision
  → wait for AgentSessionEvent.prompted
  → deliver user's answer back to Orca
```

Useful signals for elicitations:

- `auth`: user must complete account linking/authentication before continuing.
- `select`: user should choose from provided options, while free-text replies may still be possible.

### `response`

A final successful result. This normally completes the session.

Examples:

- `Implemented the change and opened PR ...`
- `I investigated the issue and found ...`
- `Stopped as requested. No further changes will be made.`

Use `response` when work is complete or when a stop request has been safely honored.

### `error`

A final or blocking failure. This normally puts the session into an error state.

Examples:

- no authorized repo matched the issue,
- no execution host is available and queueing is not allowed,
- Orca worker failed to start,
- worker crashed,
- credentials or permissions are missing,
- the router cannot reconcile uncertain execution state.

Keep public error bodies safe and concise. Store detailed traces internally with secret redaction.

### `prompt`

`prompt` is user-generated input, not an agent output type. Linear may represent user follow-ups or elicitation answers as `prompt` Agent Activities, which trigger `prompted` webhooks.

The Orca router must ingest `prompt` activities but must not create `prompt` activities itself.

## Human-to-agent signals

Signals are optional metadata attached to Agent Activities.

### `stop`

A `stop` signal on a user `prompt` means the agent must halt work immediately.

Required behavior:

1. Stop accepting further side effects for the session.
2. Fence or stop the Orca worker.
3. Persist cancellation state.
4. Emit a final `response` or `error` confirming the stopped state.

Suggested Orca mapping:

```text
prompted + signal=stop
  → fence/stop orchestration worker or terminal
  → mark router session canceled
  → emit response: "Stopped as requested ..."
```

After a stop signal, do not continue code changes, API writes, deployments, or other side effects for that session.

## Recommended router internal lifecycle

Linear's session states are intentionally high-level. The router should keep a more detailed internal lifecycle for reliability and recovery.

```text
received
  → accepted
  → routed
  → queued
  → starting
  → running
  → awaiting_input
  → resuming
  → completing
  → completed
```

Failure/cancellation branches:

```text
routing_failed
host_unavailable
start_failed
worker_failed
canceled
stale_or_reconcile_needed
```

Suggested mapping to Linear activities:

| Router state | Linear output |
| --- | --- |
| `accepted` | `thought` |
| `queued` | `thought` |
| `starting` | `action` |
| `running` | `thought` or `action` |
| `awaiting_input` | `elicitation` |
| `resuming` | `thought` |
| `completed` | `response` |
| `canceled` | `response` or `error` |
| failure states | `error` |

## Orca mapping

The router should persist a durable mapping between Linear state and Orca execution state:

```text
Linear workspace id
+ Linear issue id
+ Linear agentSession id
    ↔ router job/session id
    ↔ Orca project/repo id
    ↔ execution host id
    ↔ Orca worktree id
    ↔ Orca orchestration run/task/dispatch id or terminal handle
```

For first implementation, prefer Orca supervised orchestration when possible because it provides lifecycle operations and bounded reads:

```bash
orca orchestration worker-start \
  --spec "<initial prompt>" \
  --worktree new-top-level \
  --repo id:<repoId> \
  --name linear-<issue-key> \
  --agent <agentId> \
  --json
```

A simpler direct worktree launch is also possible:

```bash
orca worktree create \
  --repo id:<repoId> \
  --name linear-<issue-key> \
  --linear-issue <issue-key-or-url> \
  --agent <agentId> \
  --prompt "<initial prompt>" \
  --setup run \
  --json
```

For follow-ups, reuse the persisted worker mapping. Do not route solely from the issue key because a single issue may have multiple Agent Sessions.

## Reliability requirements

- Treat Linear webhooks as at-least-once delivery.
- Deduplicate by delivery id and/or event identity.
- Persist events before acknowledging webhooks.
- Preserve message ordering per `agentSessionId`.
- Do not launch duplicate Orca workers after retries or connector crashes without reconciliation.
- Persist outgoing Linear response intent before posting where possible.
- Track delivery status to avoid duplicate visible responses.
- Reconcile Orca worker/worktree state after router or connector restart.
- Distinguish queued, running, awaiting input, completed, failed, and canceled in the router's own store.

## Minimum v1 support checklist

- [ ] `AgentSessionEvent.created`
- [ ] `AgentSessionEvent.prompted`
- [ ] `thought` acknowledgement within Linear's expected startup window
- [ ] `action` for major Orca operations
- [ ] `elicitation` for user questions/approvals
- [ ] `response` for final success or safe cancellation
- [ ] `error` for final/blocking failure
- [ ] Ingest user `prompt` activities without emitting them
- [ ] Handle `stop` signal
- [ ] Maintain Linear session → Orca worker mapping
- [ ] Preserve per-session prompt ordering
- [ ] Expose safe `externalUrls` for session/worktree/PR links where available
- [ ] Reconcile after retries, duplicate webhooks, and restarts
