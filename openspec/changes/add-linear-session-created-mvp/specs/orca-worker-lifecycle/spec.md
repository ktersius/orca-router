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

### Requirement: Mandatory task worktree placement
Every attempt, including live integration checks, SHALL use a new isolated task worktree on a non-default task branch. It SHALL NOT execute in the primary checkout or on main, master, or the configured default branch. If placement cannot be established, the router SHALL NOT dispatch task instructions to the worker.

#### Scenario: Placement is safe
- **WHEN** a new attempt is launched
- **THEN** its task executes only in its new worktree on a non-default branch

#### Scenario: Placement is unsafe or unverifiable
- **WHEN** the target resolves to the primary checkout, a prohibited branch, or unverified placement
- **THEN** task execution is not started and the placement failure is reported

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

### Requirement: Persistent execution deadline
The router SHALL default to a configurable 30-minute deadline measured from confirmed worker startup and persisted across restarts. Deadline expiry SHALL request attempt-scoped cancellation, not imply termination. Recorded blocked attempts SHALL be exempt from automatic deadline cancellation. Uncertain stopping SHALL retain the slot; confirmed timeout stopping SHALL release it without cleanup or retry.

#### Scenario: Router restarts during execution
- **WHEN** the router restarts with an active worker whose deadline has passed
- **THEN** it reconciles that attempt and requests cancellation if still executing rather than starting a fresh 30-minute allowance

#### Scenario: Worker is blocked
- **WHEN** the worker has been explicitly recorded as blocked awaiting clarification
- **THEN** passage of its execution deadline alone does not cancel or close it

#### Scenario: Worker remains busy past the cap
- **WHEN** an unsettled, non-blocked worker reaches the deadline even with recent liveness signals
- **THEN** the router persists timeout intent and requests cancellation of that exact attempt
- **AND** it releases the slot only with positive stopping evidence

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

### Requirement: Operator-approved execution environment
The MVP SHALL permit reuse of the current operator-approved runtime environment and credentials without requiring a dedicated identity or new branch protections. Worker instructions SHALL prohibit merging and deployment; documentation SHALL NOT claim those restrictions are technically enforced. Normal tasks SHALL permit authorized task-branch pushes and review PRs.

#### Scenario: Existing credentials are reused
- **WHEN** the operator configures the current execution environment for the pilot
- **THEN** a new restricted identity is not a rollout prerequisite
- **AND** the worker receives no-merge/no-deploy instructions without a false sandbox guarantee

### Requirement: Scoped live-test instructions
Existing repositories SHALL be permitted for explicitly approved bounded integration tests. Test workers SHALL receive a small explicit instruction set prohibiting pushes and PR creation unless that particular test requires them. The mandatory task-worktree placement SHALL apply to all such tests.

#### Scenario: Basic lifecycle test
- **WHEN** a live test exercises launch, question, completion, or cancellation without needing publication
- **THEN** its instructions prohibit pushes and PR creation and confine task work to a new isolated worktree

### Requirement: Verified integration gate
Live execution SHALL remain disabled until the selected CLI/runtime/backend combination demonstrates cross-container coordinator operation, structured question correlation, explicit outcomes, cancellation, delivery replay, and recovery. Missing support SHALL stop rollout rather than trigger terminal scraping.

#### Scenario: External coordinator cannot consume structured questions
- **WHEN** verification cannot establish the required coordinator identity or message contract
- **THEN** live execution is not enabled and the unsupported contract is reported for design review
