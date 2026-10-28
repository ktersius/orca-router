# Spec Delta

## Purpose

Connect explicit Linear agent-session requests to bounded execution attempts and give users truthful feedback without supporting an interactive clarification conversation.

## ADDED Requirements

### Requirement: Authenticated durable acceptance
The router SHALL validate raw-body HMAC, timestamp freshness, payload shape, and allowed workspace before accepting supported events. It SHALL durably record acceptance before returning HTTP 200 within five seconds, without waiting for agent execution.

#### Scenario: Valid creation
- **WHEN** an authorized, correctly signed session creation event arrives
- **THEN** the router persists it and returns HTTP 200 within five seconds independently of worker startup

#### Scenario: Invalid or unauthorized event
- **WHEN** signature, freshness, required fields, or workspace validation fails
- **THEN** the router returns a non-success response and performs no worker launch or Linear write

#### Scenario: Persistence fails
- **WHEN** the router cannot durably record an otherwise valid event
- **THEN** it returns a retryable non-success response and does not launch work

### Requirement: Trusted-workspace initiation
The router SHALL trust members of the configured Linear workspace who can mention or delegate to the app to initiate work without a separate initiator allowlist. This trust SHALL NOT broaden the configured execution target or bypass webhook validation.

#### Scenario: Workspace member starts work
- **WHEN** a valid creation event originates from a member of the configured workspace
- **THEN** admission does not require a separate per-user authorization list
- **AND** the configured repository and backend remain fixed

### Requirement: Idempotent session creation
The router SHALL use workspace and agent-session identity to identify an attempt and deduplicate delivery identity. Retries or repeated creation notifications for the same session SHALL NOT start additional workers or intentionally repeat recorded feedback.

#### Scenario: Repeated creation
- **WHEN** a creation event is redelivered or the same session is reported under another delivery identity
- **THEN** the router acknowledges the event using the existing attempt without creating another worker

### Requirement: Single-attempt admission
The router SHALL admit at most one execution attempt at a time against the operator-configured repository and backend. Blocked or uncertain attempts SHALL occupy the slot. Additional sessions SHALL be declined, not queued, while the service remains responsive to control events.

#### Scenario: Concurrent new sessions
- **WHEN** multiple authorized sessions arrive while the slot is free
- **THEN** only one reserves the slot and the others receive a Linear error explaining that they were not started or queued

#### Scenario: Blocked attempt
- **WHEN** an attempt remains blocked and another session arrives
- **THEN** the new session is declined while stop signals and monitoring remain available

### Requirement: Prompt acknowledgment activity
For an admitted creation event, the router SHALL initiate a truthful Linear thought activity early enough to deliver it within ten seconds of receipt under healthy API conditions, independently of worker startup. It SHALL record API failures and SHALL NOT claim acknowledgment was delivered when it was not.

#### Scenario: Slow worker startup
- **WHEN** worker startup takes longer than ten seconds but Linear is reachable
- **THEN** the session receives an acceptance/preparation thought within ten seconds without a false claim that execution has begun

### Requirement: Outcome feedback
The router SHALL report confirmed success with a response activity and confirmed failure with an error activity. PRs SHALL be required only when requested by the task. Feedback SHALL include relevant results and disclose known partial branches or PRs, without exposing secrets or unrestricted logs. It SHALL NOT change the Linear issue workflow state.

#### Scenario: Success without a PR request
- **WHEN** a worker explicitly succeeds on a task that did not request a PR
- **THEN** the router reports the result without imposing a PR requirement

#### Scenario: Requested PR is missing
- **WHEN** a worker claims success for a task requesting a PR but provides no PR evidence
- **THEN** the router does not publish a successful completion and reports the unmet requirement

### Requirement: Blocked feedback without elicitation
On a correlated worker question, the router SHALL record the attempt as blocked and send a final Linear error describing the blocker, worker preservation, and lack of reply-driven resumption. It SHALL NOT emit elicitation, answer the question automatically, or close the worker solely because it is blocked.

#### Scenario: Clarification required
- **WHEN** the worker asks a structured question requiring human input
- **THEN** Linear receives the blocker explanation while the worker and execution slot remain preserved

### Requirement: Stop-only prompted handling
The router SHALL process authorized prompted stop signals for mapped unsettled attempts, including blocked attempts. It SHALL NOT launch or resume workers for ordinary follow-ups. Unsupported follow-ups SHALL receive a clear limitation notice without changing execution state or replacing an existing terminal result.

#### Scenario: Ordinary reply to a blocked session
- **WHEN** a user replies with clarification
- **THEN** the worker is not resumed and the user is informed that Linear-driven resumption is unsupported

#### Scenario: Stop requested
- **WHEN** a stop signal targets a running or blocked attempt
- **THEN** the router persists cancellation intent and requests cancellation of that exact attempt
- **AND** it confirms cancellation to Linear only after stopping is established

#### Scenario: Unknown or settled session
- **WHEN** a stop signal has no unsettled mapped attempt
- **THEN** it does not affect another worker or launch a replacement

### Requirement: Execution-timeout feedback
On execution deadline expiry, the router SHALL send a Linear error distinguishing confirmed stopping from unknown stopping. It SHALL disclose known partial artifacts, avoid automatic retry, and SHALL NOT claim the worker stopped without positive evidence. Recorded blocked attempts SHALL retain their preservation policy.

#### Scenario: Deadline cancellation confirmed
- **WHEN** a worker exceeds its execution deadline and stopping is confirmed
- **THEN** Linear receives a timeout error explaining preservation of its worktree/conversation and lack of automatic retry

#### Scenario: Deadline cancellation uncertain
- **WHEN** a worker exceeds its execution deadline but stopping cannot be confirmed
- **THEN** Linear receives an error saying it may still be running and requires operator inspection
- **AND** the execution slot remains occupied

### Requirement: Durable outgoing feedback
The router SHALL persist feedback intent and observed delivery status. It SHALL reconcile uncertain API writes rather than blindly repost them, and SHALL expose unresolved delivery failures for operator inspection.

#### Scenario: API response is lost
- **WHEN** an activity may have been posted but its result is unknown
- **THEN** the router preserves the uncertainty and reconciles or requires manual intervention before retrying that write
