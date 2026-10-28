# Design

## Context

See proposal.md for motivation and the two delta specs for behavior. This repository currently contains documentation and agent tooling, not application code or tests. The broad lifecycle reference includes features deliberately excluded here. This change defines the accepted two-container MVP topology.

Read-only inspection of the installed Orca CLI establishes documented commands, not a proven deployed integration:

- `worker-start --spec ... --worktree new-top-level --repo ... --agent ... --run ... --json` exposes supervised launch receipts; failed or outcome-unknown launches can leave residual resources.
- Remote-runtime access is available through configured environments/pairing. `--on` selects a worker server relative to the current Run home; it is not interchangeable with connecting the CLI to the intended runtime.
- Workers use their injected `ask` command for questions and `send --type worker_done --task-id ... --dispatch-id ... --outcome succeeded|failed` for completion.
- `check --run ... --wait --types ... --json` returns whole FIFO delivery batches, not only the requested wake types. Deliveries replay until acknowledged; keepalives go to stderr.
- Ask timeouts leave the question pending; worker instructions require resuming its message ID. Settled workers can be retained rather than released.

Exact envelopes and external coordinator identity have not been captured. No live workers were launched during planning.

## Goals / Non-Goals

**Goals:** keep HTTP handling responsive; make execution authority, local state, and visible Linear status distinct; implement durable transitions without a distributed broker; validate the actual deployment boundary before relying on it.

**Non-Goals:** implementing Orca, inventing a callback protocol, building a complete simulator, or claiming exactly-once external side effects. See proposal.md for excluded product features.

## Decisions

### Two containers, one router application

The router image contains Node.js/TypeScript and a compatible pinned Orca CLI. The separate execution image/runtime owns repositories, worktrees, coding agents, and their credentials. The router invokes CLI child processes with argument arrays and a configured authenticated runtime connection. No shell interpolation of event text and no fallback to local execution. Runtime ports stay private; only the webhook ingress is publicly exposed through operator-managed HTTPS.

Alternative: hosted receiver plus connector. Rejected as unnecessary infrastructure. Putting runtime and agents in the router image would unnecessarily couple execution dependencies and credentials.

### Verify the contract before building around it

The first implementation gate is a bounded, explicitly approved cross-container experiment using an existing operator-selected repository and one configured backend. Each test uses a new isolated task worktree and a small explicit instruction set; prohibit pushes and PR creation unless that particular test requires publication. No disposable repository is required. Prove external Run/coordinator identity, launch receipts, questions with verifiable attempt correlation, completion, stop semantics, pending-ask timeout behavior, delivery replay/ack, and restart lookup. Record supported versions and sanitized actual JSON samples. Verify settled retention separately from blocked preservation.

Do not invent JSON field names in advance. If correlation or external coordinator consumption cannot be proven, stop rollout and return for a design decision; do not scrape terminal text. Backend/repo identifiers are deployment settings, not planning-time defaults.

### Trusted workspace and mandatory placement

Trust members of the configured Linear workspace who can mention/delegate to the app; no additional initiator allowlist is required. Webhook authentication and workspace validation still apply, and task text cannot broaden the configured target.

Always create a new isolated task worktree on a non-default task branch, including for integration checks. Never dispatch task work into the primary checkout or main/master/configured default branch. Validate launch placement before delivering task instructions; if the CLI cannot support that safety boundary, stop the contract gate rather than fall back to an existing checkout. Basing a task branch on the default branch is allowed; executing task changes on the default branch is not.

### Small durable ledger and single-slot admission

Use SQLite on a persistent router volume, with transactional admission and uniqueness constraints. Conceptually retain deliveries, workspace/session attempt identity, execution state/receipts, processed Orca message identities, and outgoing activity intents/status. Specific table and interface layouts remain implementation details.

Execution states: `accepted → starting → running → completed|failed`; branches include `blocked`, `canceling → canceled|timed_out`, and `reconcile_needed`. Blocked and uncertain attempts hold the slot. Known no-launch failure, explicit settlement, or confirmed stop releases it. One router replica only; the deployment must not scale this stateful service horizontally.

A new session is not a durable backlog entry: atomically admit it or persist its busy rejection and response intent. Acknowledge valid persisted ingress within five seconds, then run CLI/API work asynchronously in the same service. Prioritize the initial truthful thought so a slow launch does not miss Linear's ten-second acknowledgment expectation.

### Separate execution from Linear interaction state

Store execution state independently from Linear feedback. A worker question sets local `blocked` and creates a final Linear error explaining that the worker is preserved and replies will not resume it. The router neither answers nor closes the worker. The pending ask must remain pending across timeout; prove this behavior and instruct the worker to resume the same ask rather than continue autonomously.

An operator may resolve work directly in Orca. Continue monitoring for settlement to release the slot, but do not automatically reopen a Linear interaction already ended by the blocker error. Stop control remains valid for the preserved worker. For unsupported ordinary follow-ups, use a concise limitation notice without replacing the prior terminal Agent Activity result; an ordinary issue comment is an acceptable implementation option where the app has suitable permission. Deduplicate such notices and never feed their content back as execution triggers.

### Worker instructions and result evidence

Build a bounded task specification from the event's context plus operator constraints; all Linear guidance, text, links, and attachments remain untrusted task data. Preserve Orca's injected identity/command preamble. Require structured ask for blockers, explicit worker_done outcome, acceptance evidence, and a PR URL only if the task requested a PR. Do not require PR creation for all code changes. Never parse natural-language completion sentiment as an outcome. A claimed success missing required PR evidence is recorded as unmet acceptance, not published as success.

Reuse the current operator-approved runtime environment and credentials. A dedicated restricted identity and new branch protections are not rollout requirements. Permit normal task-branch pushes and review PR creation; instruct workers not to merge or deploy. These are behavioral constraints, not technically enforced guarantees of this MVP; the operator accepts the authority of existing credentials. Disclose partial artifacts in failed/blocked results.

### Process whole deliveries, then acknowledge

Validate each Orca event against the expected active attempt and persist its decision and Linear response intent before acknowledging the whole batch. Replayed identities produce no repeated transitions. Unknown/unrelated messages are retained for inspection without affecting the slot. Structured escalation requests intervention; it does not by itself prove execution stopped. Monitoring timeouts and keepalives are not lifecycle outcomes.

### Conservative stop and restart handling

Persist cancellation intent before issuing the attempt-scoped stop operation; serialize it with startup. If no launch was issued, suppress it. Otherwise reconcile startup and stop the resulting Dispatch. Do not announce cancellation until the execution host confirms stopping. Loss of contact leaves `reconcile_needed`, preserving slot and stop intent.

Persist launch intent before invoking the CLI. On restart, use receipts/request lookup where supported by the verified version. If a crash left launch outcome unknown, do not blindly rerun startup. Recovery can require manual inspection. Confirmed settlement frees the slot; retention uses Orca's supported retention/accounting operation, not terminal closure.

### Execution deadline, not an inactivity detector

Default to a configurable 30-minute elapsed deadline from confirmed worker startup. Persist the start time/deadline so restart or lost contact does not reset the allowance. Live-state inspection/heartbeats inform diagnosis but do not extend the cap or prove useful progress. This is not a timer starting when the operator configures the service.

Once an attempt is recorded as blocked, exempt it from automatic deadline cancellation; retain the pending question and worker. If an operator resumes it directly in Orca, keep the prior Linear interaction closed and monitor for settlement; the MVP does not automatically re-arm the deadline after recorded blocking. Manual stop remains available.

When an executing attempt reaches its deadline, persist timeout reason and cancellation intent and invoke the same attempt-scoped cancellation path as user stop. Confirmed stopping records `timed_out`, sends a final Linear error with known partial artifacts, and frees the slot while preserving resources. Unknown stopping records `reconcile_needed`, sends an error explicitly saying the worker may still run, and retains the slot. Do not automatically retry. On restart reconcile an expired attempt before deciding whether it still needs cancellation; an already-settled attempt is not stopped again. Treat result/deadline races according to confirmed lifecycle evidence rather than overwriting settlement based solely on timer delivery.

### Minimal outbound delivery handling

Persist Linear activity intent before calling the companion-owned API client; record success, known failure, or uncertainty. Bounded retries are appropriate only for known safe transient failures. A lost write response requires read-back/reconciliation or manual intervention rather than a blind repost. No claim of exactly-once Linear posting. Expose redacted state and response failures through structured logs/operator inspection, without building a dashboard.

### Lightweight testing, details deliberately deferred

Test router decisions with the smallest useful controlled inputs, and test actual CLI argument construction/JSON parsing against verified samples. Do not prescribe a fake adapter interface, fixture format, test library, or a clone of the CLI. Separately run opt-in real-Orca and Linear sandbox tests with bounded cost/time. Existing repositories are acceptable; always use new task worktrees and explicit no-push/no-PR test instructions unless publication is part of the approved test. Normal tests must not launch paid workers or write to Linear.

## Risks / Trade-offs

- [Remote CLI features do not establish usable external coordinator identity] → Treat the cross-container contract experiment as a blocking gate.
- [A model ignores reporting instructions or continues after ask timeout] → Exercise the chosen backend; require positive lifecycle evidence, preserve unknown state, and block live rollout if question behavior is unsafe.
- [Blocked workers keep the service busy indefinitely] → Document manual resolution/stop and retain identifiers; no automatic timeout cleanup.
- [Final Linear error coexists with a live blocked Orca attempt] → Make the distinction explicit in feedback and operator state; do not imply Linear resumption is available.
- [Retained workers consume resources or humans manually restart them] → Document ownership and resource costs; no automatic reuse of settled authority.
- [SQLite loss could lose worker authority] → Persistent volume, backups, single replica, and operator reconciliation before starting with lost state.
- [Existing credentials permit more than worker instructions allow] → Explicitly document operator acceptance and no-merge/no-deploy instructions; do not promise credential enforcement. Keep runtime inaccessible publicly and isolate secrets by container.
- [Linear/API downtime breaks timing promises] → Record pending/uncertain response status, report degraded readiness, and avoid claiming delivery without evidence.

## Migration Plan

There is no application or production data to migrate. Implement the contract gate first, then the ledger and router, then package the router image. Manually configure the allowed workspace/repository/backend, Linear app token and webhook secret, runtime connection, HTTPS ingress, persistent state, and accepted existing execution permissions. See `docs/deployment-environment.md` for the surrounding Proxmox VM, private runtime ingress, and planned Cloudflare webhook path; these are not live-verified by this plan. Pin and document the verified CLI/runtime versions.

Run a sandbox smoke test before enabling real sessions. On rollback, disable new admission first; preserve the database and execution resources and reconcile any live/blocked/uncertain Dispatches. Stopping the router container is not proof that remote workers stopped. Do not automatically delete volumes, worktrees, conversations, or branches.

## Open Questions

Deferrable implementation details: HTTP framework, SQLite binding/migration mechanism, precise local inspection command shape, lightweight test tooling, and backend-specific evidence formatting. Exact CLI JSON names will be determined by the mandatory contract experiment without weakening the behavioral specs.
