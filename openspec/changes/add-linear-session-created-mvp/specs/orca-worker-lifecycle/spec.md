# Spec Delta

## Purpose

Manage one authoritative Orca worker attempt using structured lifecycle signals, preserving resources and uncertainty instead of inferring outcomes from terminal activity.

## ADDED Requirements

### Requirement: Explicit remote execution boundary
The router SHALL invoke its Orca CLI as a child process with structured arguments and an authenticated connection to the separately configured runtime. Execution target configuration SHALL come from the operator, not untrusted task content. A failed connection SHALL NOT cause fallback to local execution.

#### Scenario: Task attempts to select another repository
- **WHEN** issue content names a repository outside the configured target
- **THEN** it cannot override launch placement or broaden access

#### Scenario: Runtime is unreachable
- **WHEN** the remote runtime cannot be reached
- **THEN** the router does not execute locally or claim a worker started

### Requirement: Correlated worker launch
The router SHALL persist an attempt and launch intent before invoking worker startup, then persist available Run, Task, Dispatch, worker, and worktree identifiers. It SHALL launch in an isolated worktree and accept lifecycle authority only for the expected attempt.

#### Scenario: Message from another attempt
- **WHEN** a structured event cannot be correlated to the expected active attempt
- **THEN** it does not change that attempt's state or release its slot and is retained for inspection

### Requirement: Explicit worker reporting contract
Workers SHALL receive instructions to use Orca's injected blocking ask command for clarification and worker_done with explicit succeeded or failed outcome for settlement. The router SHALL consume documented structured messages rather than infer lifecycle state from terminal prose, silence, or idleness.

#### Scenario: Structured completion
- **WHEN** a valid worker_done identifies the authoritative Task and Dispatch with an explicit outcome
- **THEN** the router records the corresponding settlement and result evidence

#### Scenario: Quiet terminal
- **WHEN** output is silent or a read times out without an authoritative lifecycle outcome
- **THEN** the router does not mark the task completed, failed, or stopped solely on that observation

### Requirement: Preserved blocked attempts
A correlated blocking question SHALL make the local attempt blocked without closing its worker, releasing its slot, or settling it as failed. The question identity and text SHALL be preserved. Ask timeouts SHALL NOT authorize continuation or duplicate questions. Resumption through Linear SHALL remain disabled.

#### Scenario: Blocking ask times out
- **WHEN** the question remains unanswered after the ask command times out
- **THEN** the same question remains pending and the router does not resume the task or create another worker

#### Scenario: Manual resolution in Orca
- **WHEN** an operator resolves a blocked worker directly in Orca
- **THEN** the router keeps the slot occupied until explicit settlement or confirmed stopping
- **AND** it does not silently reopen the already-ended Linear interaction

### Requirement: Attempt-scoped cancellation
The router SHALL persist stop intent and prevent further router-driven task actions before requesting cancellation of the exact Dispatch. It SHALL report stopped and free the slot only on positive confirmation. Uncertain cancellation SHALL remain unresolved and occupy the slot.

#### Scenario: Stop races with launch
- **WHEN** cancellation is received while startup is in progress
- **THEN** the router prevents launch if not yet issued or reconciles and stops any resulting worker before confirming cancellation

#### Scenario: Stop cannot be verified
- **WHEN** the runtime disconnects during cancellation
- **THEN** the router preserves stop intent and uncertainty without claiming the worker stopped

### Requirement: Conservative recovery
After restart or ambiguous launch, the router SHALL reconcile persisted execution intent and receipts before another launch. An unresolved attempt SHALL occupy the slot and SHALL NOT be automatically retried as new work. Confirmed no-launch failures, settlements, and stops SHALL free the slot.

#### Scenario: Crash after launch before receipt persistence
- **WHEN** startup might have succeeded but its receipt was not saved
- **THEN** recovery checks the original operation or requires manual reconciliation rather than launching a replacement

### Requirement: Durable lifecycle consumption
The router SHALL persist correlated message decisions and resulting response intent before acknowledging a delivery. Replayed messages SHALL NOT repeat transitions or response intents. Every message in a returned batch SHALL be processed, including types outside the wait filter.

#### Scenario: Crash before acknowledgment
- **WHEN** a persisted message decision is delivered again after restart
- **THEN** it is recognized as processed without repeating its side effects

### Requirement: Retention independent of settlement
The router SHALL preserve worktrees and worker conversations using supported Orca lifecycle operations rather than automatic cleanup. Retained settled workers SHALL NOT occupy the execution slot or be authorized to continue the settled task.

#### Scenario: Successful task retained
- **WHEN** explicit success settles an attempt
- **THEN** its resources remain available for inspection while the slot becomes available for a new task

### Requirement: Restricted task authority
Workers SHALL be permitted to edit, test, push task branches, and create review PRs within authorized scope, but SHALL NOT merge or deploy. The deployment SHALL enforce these boundaries through available permissions rather than relying solely on prompt instructions.

#### Scenario: Task requests a deployment or merge
- **WHEN** task content asks for an operation beyond the pilot's authority
- **THEN** it does not gain the required permissions and the unmet request is reported rather than executed

### Requirement: Verified integration gate
Live execution SHALL remain disabled until the selected CLI/runtime/backend combination demonstrates cross-container coordinator operation, structured question correlation, explicit outcomes, cancellation, delivery replay, and recovery. Missing support SHALL stop rollout rather than trigger terminal scraping.

#### Scenario: External coordinator cannot consume structured questions
- **WHEN** verification cannot establish the required coordinator identity or message contract
- **THEN** live execution is not enabled and the unsupported contract is reported for design review
