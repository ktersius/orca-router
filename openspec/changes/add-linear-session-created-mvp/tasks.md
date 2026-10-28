# Tasks

## 1. Verify the actual Orca boundary before depending on it

- [ ] 1.1 With explicit live-test approval, document the chosen runtime/CLI versions, backend, disposable repository, and time/cost limit; verify the CLI in a separate container can authenticate, create/address a Run, and establish coordinator identity without an interactive router terminal. Stop for design review if unsupported.
- [ ] 1.2 Run bounded workers that ask a question and report succeeded/failed; capture sanitized real launch receipts and message deliveries, proving Task/Dispatch correlation and documenting exact envelopes without inventing a callback protocol.
- [ ] 1.3 Verify pending ask behavior across timeout, full-batch replay/ack, attempt-scoped stopping, settled retention, and restart/request lookup; deliver a contract report listing guarantees and limitations and block live rollout on any unmet required behavior.

## 2. Bootstrap the service and durable admission

- [ ] 2.1 Set up a minimal TypeScript/Node.js build and choose small HTTP/SQLite/test dependencies during implementation; verify clean install, typecheck, and offline test execution without a fake CLI implementation.
- [ ] 2.2 Implement and document configuration for the allowed workspace/repository/backend, runtime connection, secrets, and persistent database; verify missing/unsafe configuration fails clearly and secrets are excluded from tracked examples and logs.
- [ ] 2.3 Implement the SQLite attempt/delivery/response ledger and transactional slot reservation; verify concurrent admissions, session/delivery deduplication, persistence across restart, and blocked/uncertain slot retention with lightweight tests.
- [ ] 2.4 Implement signed raw-body webhook validation and durable HTTP acceptance; verify valid, malformed, stale, unauthorized, and persistence-failure requests, including a slow-launch case acknowledged within five seconds.

## 3. Launch and consume structured worker lifecycle

- [ ] 3.1 Implement the narrow Orca CLI integration using the verified contract and argument arrays; verify remote-only placement, recorded launch intent/receipts, adversarial task text handling, and no local fallback without recreating the CLI.
- [ ] 3.2 Build and document the worker task/reporting instructions, preserving Orca's injected preamble; verify they require blocking ask, explicit outcomes, acceptance evidence, and PRs only when requested, and document enforceable no-merge/no-deploy controls.
- [ ] 3.3 Implement whole-batch message consumption, authority checks, deduplication, and persist-before-ack; verify replay, unrelated/invalid messages, keepalives, timeouts, and out-of-filter batch members do not create false lifecycle outcomes.
- [ ] 3.4 Implement blocked, completed, and failed transitions and supported retention; verify questions preserve the pending worker/slot, explicit settlement frees the slot, required PR evidence is checked, and terminal silence never settles work. Document manual blocked-worker ownership and resolution.

## 4. Add Linear feedback and stop-only control

- [ ] 4.1 Implement companion-owned Linear API access and persisted response intent/status; verify successful delivery, known failures, redaction, and uncertain writes requiring reconciliation rather than blind reposts.
- [ ] 4.2 Implement acceptance thoughts and busy rejection errors; verify healthy API acknowledgment within ten seconds despite slow startup and that declined sessions are not queued or launched.
- [ ] 4.3 Implement final success/failure/blocker feedback and unsupported-follow-up notices; verify no elicitation, no issue workflow-state changes, no automatic question answers/resumption, no unwanted reopening after manual Orca resolution, and disclosure of known partial artifacts. Document the intentionally different Linear and Orca blocked states.
- [ ] 4.4 Implement persisted stop intent serialized with launch and exact-Dispatch cancellation; verify stop-before-start, stop-during-start, blocked-worker stop, stale/unknown-session isolation, and disconnect uncertainty, confirming cancellation only with positive stopping evidence.

## 5. Recover without duplicate execution

- [ ] 5.1 Implement startup reconciliation of accepted, starting, running, blocked, and canceling attempts using verified receipts/request lookup where available; verify crash-after-launch-before-receipt does not start a replacement and unresolved execution retains the slot.
- [ ] 5.2 Provide minimal redacted operator inspection and a manual recovery runbook; verify an operator can identify the attempt, pending question, remote identifiers, stop intent, and uncertain response state without automatic cleanup or unsafe slot release.

## 6. Package and prove the two-container deployment

- [ ] 6.1 Package Node.js and a pinned compatible Orca CLI in the router image, keeping the execution runtime external; verify a container restart preserves SQLite state and runtime connectivity without local repository or agent credentials.
- [ ] 6.2 Document manual setup for Linear app scopes/credentials, authenticated private runtime access, HTTPS ingress, persistent volumes, one router replica, execution permission boundaries, retention costs, and rollback; verify the runbook in a disposable two-container environment.
- [ ] 6.3 With explicit approval, run bounded real-Orca integration checks and a dedicated Linear sandbox smoke test covering mention/delegation, acknowledgment timing, completion with/without a requested PR, blocking, busy rejection, stop, and restart; record evidence against both specs and verify ordinary tests require neither live API writes nor paid workers.
