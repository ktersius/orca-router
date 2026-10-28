# Proposal

## Why

Turn an explicit Linear agent mention or delegation into useful Orca-managed work, without building the full hosted router/connector architecture. Prove one real task from session creation to a visible result, with reliable blocked and stop behavior.

## What Changes

- Add a TypeScript/Node.js router that receives signed `AgentSessionEvent.created` webhooks, persists them in SQLite, and invokes the Orca CLI asynchronously in the same application service.
- Deploy the router and CLI in one container, connecting to an authenticated Orca runtime in a separate container containing repositories and coding agents. Setup is manual.
- Configure one allowed Linear workspace, repository, and backend. Admit one attempt at a time; decline busy sessions rather than queue them. Keep serving webhooks while busy.
- Use structured Orca questions and explicit worker outcomes, not terminal scraping. A blocked attempt preserves its worker and occupies the slot; send a final Linear `error` explaining that replies will not resume it.
- Handle `prompted` only for stop control, not conversation resumption. Report confirmed success/failure/cancellation with Linear Agent Activities. Require a PR only when requested by the task; permit branch pushes and PR creation for review, but not merging or deployment.
- Preserve worktrees and conversations after settlement. Reconcile ambiguous execution before freeing the slot or launching replacements.
- Keep test support lightweight; do not recreate the Orca CLI. Verify the real cross-container contract before enabling live work.

Out of scope: elicitation, clarification round trips, issue-created automation, multiple execution targets, distributed queues, automatic onboarding, automatic cleanup, merge/deploy automation, and a custom Orca callback protocol.

## Capabilities

### New Capabilities

- `linear-agent-sessions`: Authenticated event acceptance, one-attempt admission, session feedback, and stop-only follow-ups.
- `orca-worker-lifecycle`: Structured worker launch, questions, outcomes, cancellation, retention, and uncertain-execution reconciliation.

### Modified Capabilities

None. The repository has no existing main specs.

## Impact

This is greenfield application work: no source, test suite, or container implementation exists yet. Adds a Node.js application, SQLite persistence, router image, configuration/runbook, and narrow integrations with Linear's Agent Activity API and Orca's CLI. No Orca source changes or Linear issue workflow-state changes are intended.

This OpenSpec change is the canonical MVP plan. `docs/linear-agent-lifecycle.md` describes a broader lifecycle; this change intentionally excludes elicitation and ordinary prompted resumption.

Cross-container coordinator identity, exact JSON envelopes, cancellation guarantees, and backend question behavior remain verification gates—not established capabilities of this application. If unsupported, stop and revisit the design rather than fall back to terminal heuristics.
