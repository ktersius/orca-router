# Design

## Context

See proposal.md for motivation. The repository has no router application or integration harness. The MVP's tasks 1.1–1.3 already demand external coordination evidence; this change gives that exercise its own planning boundary rather than duplicating a production adapter.

`docs/deployment-environment.md` records a homelab Compose runtime pinned in source to Orca 1.4.224, with Pi and Codex, on a Proxmox development VM. That source snapshot is not the live version. CLI help inspected in the managed agent session is not proof of the target runtime's supported schema. `GLOSSARY.md` distinguishes blocked attempts from settled retained workers.

## Goals / Non-Goals

**Goals:** establish reproducible commands and observed JSON for an independent program to own/address a Run and supervise workers; identify unsupported boundaries before implementation; leave a bounded, accounted-for set of test resources and sanitized evidence.

**Non-Goals:** router implementation, Linear/tunnel setup, runtime upgrades, broad test infrastructure, production-container restart, model credential extraction, or undocumented direct database/API access.

## Decisions

### Agent-operated experiment, externally identified caller

The agent runs the experiment, but the tested coordinator must not borrow the agent's Orca-managed terminal/session/Dispatch identity. Require a separate ephemeral CLI-only client container for a successful cross-container verdict with an isolated HOME/configuration, no runtime home/workspace mounts, no Docker socket, and only the approved runtime endpoint/grant. The operator can launch the client container; it need not control Docker from within itself.

Inspect the version-matched guide to determine identity inputs rather than guessing environment variable names. Document the client launch method and relevant identity-variable names without values. Preserve intentional remote authentication, and do not accidentally substitute a local runtime. Separate CLI process invocations must address the same Run using supported explicit identifiers or private client state; a long-lived interactive coordinator agent is not an acceptable hidden dependency.

A process-only sanitized caller can help diagnose behavior but does not prove cross-container access. Mark partial results accordingly. Unsupported independent coordinator identity is a valid negative finding, not permission to invent one.

### Small sequential scenario matrix

The agreed target repository is `orca-router`. Inspect the runtime and select an existing configured/authenticated backend, preferring Pi when it supports supervised coordination. If none is ready, report the prerequisite; do not enroll providers or alter credentials. Confirm the runtime endpoint and private access before mutations, and wait for an explicit apply request before live execution.

Budget: at most three sequential worker launches, five minutes per attempt from startup initiation (including question tests), and a 30-minute total live window beginning with the first live Run/resource mutation. No parallel launches or automatic retries. Do not begin a launch that cannot fit within the remaining window. Existing credentials are acceptable; keep provider use within approved quota/budget. At either time limit, request stopping of any unsettled test-owned attempt; killing the CLI does not prove a remote stop. If stopping is uncertain, halt new launches and report the unresolved worker. Safety accounting after a limit is not permission for more test work.

Settle or stop each test attempt before the next launch. Leave no intentionally running or unanswered test worker, but preserve worktrees/conversations for inspection. The spike explicitly cancels its question test within these limits; it does not change the MVP's policy of preserving blocked workers without automatic timeout cancellation.

All task execution must use new isolated non-default task worktrees, never the primary checkout or main/master/default branch. Verify that the CLI can establish safe placement before task instructions reach the worker; if this cannot be guaranteed, report a blocking gap instead of proceeding unsafely. A task branch may be based on the default branch. Keep task instructions narrowly bounded with no push, PR, merge, deploy, or unrelated work.

Use a small core proof, not an exhaustive lifecycle suite:

| Core check | Positive evidence needed |
| --- | --- |
| External Run ownership | Separate container creates/addresses its own Run across independent CLI commands |
| Safe launch | Receipt and placement evidence establish a new isolated non-default task worktree |
| Question | Worker uses its injected blocking ask; caller receives a structured question correlated to that attempt |
| Completion | An explicit worker-originated completion outcome settles the expected attempt; success is sufficient for the core result check |
| Cancellation | Attempt-scoped stop demonstrates positive stopping evidence, or explicitly identifies an uncertain outcome without claiming success |

Delivery replay/acknowledgment, pending-question timeout behavior, a separate explicit failed outcome, supported retention inspection, and caller-restart recovery are follow-ups only if time and launches remain. Preserve resources regardless; optional retention testing is about proving Orca's accounting interface, not permission to close conversations. Missing replay/recovery/retention evidence remains outstanding for the MVP gate. Do not create a larger harness merely to complete the matrix.

Do not fake terminal reports to make a scenario pass. Cancel the question scenario after capturing its core evidence, within its five-minute limit. No Linear resumption is needed. Late events, unsupported fields, and loss of contact must be described honestly. Unsupported branches stop with evidence: no runtime upgrade, invented identity/callback protocol, or workaround implementation. Do not deliberately interrupt or restart the shared runtime to simulate a crash.

### Evidence instead of premature interfaces

Write one concise report at `docs/verification/external-orca-coordination.md`, combining integration instructions and evidence. Include tested versions, caller/access prerequisites, command templates, exit codes, sanitized actual JSON, observed identity/correlation fields, stdout/stderr behavior, resource disposition, and verified/unsupported/unverified verdicts. Separate core findings from optional checks and explicitly list remaining MVP-gate gaps. Label documented-only behavior and inference; do not claim compatibility beyond tested versions.

Raw responses may contain secrets; keep them in protected temporary storage and publish only reviewed excerpts. Pseudonymize identifiers consistently so correlations remain understandable without exposing real task content or runtime grants. Do not create a second integration-contract document duplicating the report.

Optional helpers are disposable shell/Python snippets, not a production abstraction or reusable fake CLI. Exact helper language and evidence layout can be chosen during execution. No delta specs are added because this is proof/documentation work, not a new application contract.

### Outcome and handoff

Report the core verdict separately from the broader MVP gate verdict. A verified core proof is useful but does not imply replay, retention, or recovery have been tested; any missing mandatory MVP evidence keeps that gate blocked. A negative/partial result still completes useful investigation when reproduced and documented. Unsupported required behavior returns to the user for a design decision; no terminal scraping, runtime upgrade, new callback protocol, or broader authority is an automatic fallback.

Do not check off MVP tasks or edit KUB-12 as a side effect of the spike. Hand off findings with references so those changes can be reviewed separately.

## Risks / Trade-offs

- [Managed-session identity leaks into the client] → Isolate client configuration and document the absence of inherited caller identity; do not confuse successful remote authentication with coordinator independence.
- [Runtime version differs from the inspected guide] → Record actual versions and obtain version-matched help before live operations.
- [Agent ignores ask/reporting instructions] → Preserve actual evidence; treat the backend as unverified rather than repair messages on its behalf.
- [CLI timeout leaves a worker running] → Track exact attempt receipts, stop only test-owned attempts under approved limits, and never infer process death from client exit.
- [Credential-bearing output is captured] → Protect raw evidence, redact before publication, and never commit pairing URLs/codes or provider credentials.
- [Live tasks affect an existing repository] → New non-default worktrees, narrow instructions, no publication, and no destructive cleanup of user resources.

## Migration Plan

No application migration or deployment. Begin with read-only contract discovery, obtain live-execution approval, then create the isolated client and sequential test-owned resources. Stop if access or capability prerequisites are absent. Retain test worktrees/conversations and list their disposition; temporary client/harness removal is not worker cancellation. Dispose of private client credentials through the supported mechanism when no longer needed, without affecting other paired clients. Do not remove shared volumes or runtime state.
