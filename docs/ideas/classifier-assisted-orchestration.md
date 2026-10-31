# Classifier-assisted orchestration

Status: exploratory idea, not approved implementation scope.

## Motivation

The router can handle lifecycle events deterministically without understanding every possible worker response. Linear agent-session event types provide a structured lifecycle, although delivery duplication, ordering, retries, and recovery still require explicit handling. Orca's structured messages identify questions and reported outcomes, while their free-text content can describe unexpected situations.

The potential value of a classifier is at this boundary: interpreting a worker's question or escalation to recommend an appropriate next step. We do not yet know whether enough useful, recurring cases exist to justify it. Learn from actual feedback as the router evolves rather than prescribing a taxonomy in advance.

## Possible approach

Keep a deterministic supervisor responsible for state, permissions, correlation, deadlines, and side effects. At an explicitly permitted judgment point, provide a classifier with bounded context and a small set of allowed choices. Treat its result as advice, validated by code before any action.

```text
Structured event
      |
      v
Deterministic lifecycle handling
      |
      +--> Known mechanical case --> Fixed policy
      |
      +--> Permitted judgment point --> Classifier recommendation
                                               |
                                               v
                                    Deterministic policy gate
                                               |
                              +----------------+----------------+
                              |                |                |
                              v                v                v
                         Fixed action    Coordinator agent   Human / hold
```

Possible future distinctions include:

- A missing product or permission decision that needs a human.
- A technical coordination problem that merits invoking a full coordinator agent.
- A question covered by an explicit, already-authorized mechanical response.
- Insufficient or ambiguous evidence, requiring a hold rather than a guessed action.

These are hypotheses, not an agreed classification scheme. Categorization is only operationally useful if it changes an authorized next step. If every case still needs human attention, classification may offer little beyond reporting.

## Candidate: Jev

Pi exposes TypeSafe's Jev as a classifier accepting JSON state and typed questions. It supports choices among named options, yes/no questions, and ordered scores, with probabilities or confidence information depending on the answer type. Multiple questions can be included in one request.

Unlike a full agent, the classifier does not independently inspect repositories or execute tools. The caller supplies the evidence and interprets the result. This makes it a plausible inexpensive adviser for deciding whether more capable coordination is warranted, not a substitute for that coordination.

At exploration time, the session catalog exposed Jev through OpenRouter, including `typesafe/jev-1.14` and `~typesafe/jev-latest`, with a listed input price of $0.042 per million tokens and no output-token charge. This is catalog information, not a measured cost, latency, availability, or accuracy guarantee. Recheck the provider contract and pricing before evaluation; credentials available in an interactive Pi session do not establish access from the deployed router.

Capability references inspected:

- Pi documentation: `docs/models.md`, section “Use classifier models”.
- Pi documentation: `docs/codemode.md`, section “Classify”.
- Pi example: `examples/extensions/jev-router.ts`, which classifies task complexity to select a model.

These paths are relative to the installed Pi package, not this repository.

## Boundaries

A classifier must not replace authoritative runtime evidence or deterministic policy. Do not use it to infer that a worker stopped, a launch occurred, a delivery was acknowledged, an event belongs to an attempt, or an execution deadline expired. Terminal prose and silence do not become explicit outcomes merely because a model interprets them.

Worker text and Linear content remain untrusted data. Classification must not expand permissions, bypass safe worktree placement, authorize unapproved actions, or override stop controls. Confidence scores are not demonstrated correctness rates; thresholds require evaluation. Errors, missing answers, and uncertain classifications need a defined conservative fallback.

Recognizing that an answer appears in the task is not the same as reliably producing that answer. Routing questions is a narrower initial candidate than autonomously answering them.

Classifier advice also cannot resolve an Orca external coordinator identity or access limitation. The external-coordination verification spike remains independently necessary.

## Evidence-led progression

1. Implement the agreed deterministic behavior first: a structured question marks the attempt blocked, preserves the worker, and reports the limitation without answering it.
2. Inspect actual questions and escalations, using sanitized examples with appropriate retention and access controls.
3. Identify recurring categories and whether distinguishing them would change an authorized action.
4. Evaluate a classifier in shadow mode: record recommendations without affecting worker handling. Compare against human judgments, including ambiguous and adversarial examples.
5. Only if evidence demonstrates value, propose narrowly bounded routing with explicit allowed actions, fallback behavior, budgets, and audit records.

No implementation, evaluation harness, or automated coordinator launch is authorized by this note. It does not change the MVP's blocked-worker preservation policy, reopen Linear interactions, or add tasks to the current MVP or verification spike.
