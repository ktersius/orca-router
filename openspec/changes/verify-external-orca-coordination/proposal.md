# Proposal

## Why

The router MVP depends on an ordinary external service supervising Orca through separate CLI invocations, but the exact caller identity and lifecycle contract have not been proven against the homelab runtime. Establish that boundary with a small agent-operated experiment before implementing the router or its adapter.

## What Changes

- Run a bounded CLI-only verification exercise from a separate client container connecting to the existing runtime container, without inheriting this agent session's Orca identity. A sanitized process alone is preliminary evidence, not a successful cross-container verdict.
- Core proof: independent Run/coordinator ownership across CLI calls, safe new-worktree launch, correlated structured question and explicit completion result, and attempt-scoped stopping with confirmed versus uncertain outcomes.
- Exercise delivery replay/acknowledgment, caller-restart recovery, explicit failure reporting, and retention inspection only within the remaining budget. Record unverified checks as outstanding for the MVP gate rather than expanding this spike into a harness project.
- Publish one concise evidence-backed integration report containing commands, sanitized observed JSON, findings, and remaining gaps, scoped to the actual CLI/runtime/backend versions tested. Stop unsupported branches and report limitations; do not upgrade Orca or implement workarounds.
- Permit only minimal disposable helper scripts if useful. Do not implement an HTTP service, Linear client, SQLite store, production adapter, fake Orca CLI, or a new orchestration protocol.
- Use the existing `orca-router` repository, always in new isolated non-default task worktrees. Use an already configured/authenticated backend, preferring Pi if it supports the required interface; do not automatically enroll providers or change credentials. Instructions prohibit pushes, PR creation, merging, deployment, and unrelated changes.
- Limit the live experiment to three sequential worker launches maximum, five minutes per attempt including question tests, and a 30-minute total live window. No parallel launches or automatic retries. Settle or stop each attempt before another; uncertain stopping halts further launches. Preserve worktrees/conversations but leave no intentionally running or unanswered test worker.

This change plans the experiment only. Applying it requires a new explicit apply request and confirmation of the runtime connection/access prerequisites. The repository, backend-selection policy, and launch/time limits above have been agreed; provider costs must remain within the operator's approved quota/budget. The spike's five-minute question-test limit does not change the MVP's blocked-worker preservation policy. No infrastructure upgrade, public exposure, destructive cleanup, or production-runtime restart is authorized by this proposal.

## Capabilities

### New Capabilities

None. This is an investigation/tooling/documentation spike, not new router behavior. `skip_specs: true` intentionally omits delta specs rather than pretending an unverified external API is a product capability.

### Modified Capabilities

None. Main specs are currently empty. The pending MVP's `linear-agent-sessions` and `orca-worker-lifecycle` behavior remains owned by `add-linear-session-created-mvp`.

## Impact

Planned durable output: `docs/verification/external-orca-coordination.md`, combining the integration contract and sanitized evidence in one concise report. Temporary harness files and private authentication material stay outside Git; no production deployment artifact is needed.

The experiment creates test-owned Orca Runs, attempts, conversations, and task worktrees only when separately approved. Existing runtime credentials are acceptable; do not copy provider credentials or whole runtime state into the external caller.

This spike supplies evidence for tasks 1.1–1.3 of `add-linear-session-created-mvp`. Its verified results can supply part or all of that gate's evidence, but core-proof success alone does not satisfy any omitted replay, retention, or recovery checks. Creating or completing this planning change does not check off the MVP tasks or authorize router implementation. The MVP artifacts and KUB-12 remain unchanged by this proposal.
