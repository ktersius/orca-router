# Orca Router: trimmed-down first slice

Status: proposal for discussion; not implemented.

Source: [KUB-5 — Linear → Orca Webhook Router architecture](https://linear.app/kubelord/issue/KUB-5/linear-orca-webhook-router-architecture).

## Goal

Prove one end-to-end loop: accept a task, launch an Orca worker in an isolated worktree, observe its completion, and return a result.

Start with a local simulator. Add live Linear only after the Orca lifecycle is proven. This deliberately does not implement the full [Linear agent lifecycle reference](linear-agent-lifecycle.md).

## Smallest useful scope

- One operator, one Linear workspace, one explicitly allowed repository.
- One configured Orca agent backend and one active task at a time.
- One local process both receives webhooks and invokes the Orca CLI directly as a child process; Orca must already be running on that host.
- No separate connector, broker, or worker service. After durable acceptance, acknowledge HTTP promptly and perform CLI work asynchronously in that same service.
- Manual fixture submission first; native Linear `AgentSessionEvent.created` as the proposed first live trigger.
- A fixed configuration selects workspace, repository, and backend; issue text cannot select execution targets.
- A small local durable store records accepted events, jobs, execution receipts, and outgoing response attempts.
- Fake Orca and Linear adapters by default. Real worker launches and API writes require explicit opt-in.

```text
Local fixture / later signed Linear webhook
  → validate + persist + deduplicate
  → HTTP acknowledgment after durable acceptance
  → same process applies fixed route and spawns Orca CLI
  → Orca supervised worker
  → observe explicit completion or failure
  → captured result / later Linear activity
```

The webhook receiver owns CLI invocation, lifecycle monitoring, and Linear replies. Invoke the CLI with structured argument arrays, never shell-interpolated webhook content. Do not hold the webhook request open while the CLI or coding agent runs.

No hosted/local connector split. A live development webhook would use an explicitly configured HTTPS tunnel to this process. When the host is offline, this version cannot receive or execute work; there is no always-on availability promise.

## First milestone: local proof, no Linear credentials

1. Validate the installed Orca interface for one backend: launch, receipt lookup, completion, failure, and stop. Do not assume terminal idleness means success.
2. Define a small normalized task fixture, clearly distinguished from a verified real Linear payload.
3. Process it through a fixed route and fake adapters; capture requested launch and final result.
4. Persist delivery identity and job state so submitting a duplicate does not create another worker.
5. Run one explicitly approved integration test against a disposable repository, with a time/cost limit.

Suggested minimal job states: `accepted → starting → running → completed | failed`, plus `reconcile_needed` for uncertain execution and `canceled` for a confirmed stop. Store the receipt as soon as available. After a crash during launch, do not blindly retry: block the job for reconciliation if execution cannot be established.

**Done when:** a fixture launches at most one worker under normal retries, explicit completion produces a captured result, failure is visible, and an uncertain launch is held rather than automatically repeated.

## Second milestone: narrow live Linear pilot

- Configure a dedicated development Linear app with companion-owned credentials; do not extract Orca credentials.
- Accept only authorized native session creation events, not arbitrary issue creation.
- Verify raw-body webhook HMAC, timestamp freshness, payload shape, and workspace before acceptance.
- Persist before returning HTTP 200 within Linear's five-second deadline.
- Emit a truthful `thought` acknowledgment within the expected ten-second startup window, independently of worker startup.
- Return `response` or `error` from explicit worker lifecycle events; keep output concise and redact secrets.
- Map by workspace + agent session, not issue ID alone.
- Support the user stop signal before enabling live execution. Fence further actions, stop the worker, and confirm only once stopping is established.
- Never silently discard follow-ups. Until conversation delivery is proven, reject unsupported follow-ups with a clear explanation rather than starting another worker.
- Allow only one active session; explicitly report busy/unavailable rather than building a distributed queue.

Live readiness requires sandbox verification of actual webhook shapes, acknowledgment timing, cancellation, duplicate delivery, and result rendering. Synthetic fixtures alone do not prove Linear compatibility.

## Explicitly deferred

- Hosted receiver, outbound connector, host pairing, and offline queueing.
- Multiple tenants, hosts, repositories, backends, and concurrent workers.
- Issue-created automation, label/project routing, and rule configuration UI.
- Interactive clarification/approval round trips and general conversation resumption.
- Automatic recovery of ambiguous launches and automatic replay of uncertain API writes.
- Dashboard, deployment automation, merges, and unattended risky operations.

Deferral is not permission to ignore safety: do not execute tasks requiring unsupported interaction or approval. Keep risky operations outside the pilot's permissions.

## Decisions before implementation

1. Confirm the first live trigger: native agent mention/delegation (proposed), or opt-in issue-created automation instead.
2. Pick the single repository and agent backend for the disposable integration test.
3. Choose implementation language and local store after checking the supported Orca lifecycle interface.

Next concrete task: prove the Orca launch/completion/stop contract, then scaffold only the fixture runner, fixed-route core, store, and fake adapters. No hosted infrastructure or production OAuth rollout yet.
