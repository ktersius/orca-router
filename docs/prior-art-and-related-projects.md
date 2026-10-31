# Prior art and related projects

Public-source research into deterministic coding-agent orchestration, Orca integration, and optional classifier-assisted routing.

This is a research snapshot, not a compatibility assessment or endorsement. Sources inspected included GitHub repository metadata, READMEs, an Orca issue, relevant portions of Orca Flow Kit's runner source, and portions of Symphony's specification. None of these projects was installed or executed. Repository default branches and issue status can change.

## Summary

The broad pattern is well precedented: ordinary code handles lifecycle and scheduling, agents perform task work, and a decision model can optionally help route ambiguous content.

No verified, ready-made equivalent of this project's exact **Linear agent-session webhook → isolated external CLI caller → existing Orca runtime** integration was found in this search. That is a bounded search finding, not proof that no equivalent exists. The external-coordination spike remains necessary.

| Question | Most relevant reference |
|---|---|
| Has someone scripted workflows above Orca? | Orca Flow Kit |
| Do other Orca users want deterministic orchestration? | Orca issue #11229 |
| Has someone built issue-tracker-driven coding orchestration? | OpenAI Symphony |
| Has someone used Jev for bounded routing decisions? | Aurelio Semantic Router |
| What does a broader coding-agent feedback loop look like? | Orchestrator.inc |

## Orca Flow Kit

Source: [vankhangfet/orca-sdlc-kit](https://github.com/vankhangfet/orca-sdlc-kit)

Despite its repository name, the README calls this **Orca Flow Kit** and describes general multi-step workflows on Orca ADE.

Documented features include:

- A Node.js runner above the Orca CLI.
- JSON-defined steps, agents, models, retry limits, and parallel groups.
- Review/test outcomes that route work back to an earlier step.
- Manual approval modes, persisted artifacts, and a live status page.
- Explicit worker completion handling.

### Relevance

A concrete example of ordinary code coordinating existing agents through Orca rather than asking an LLM to determine every transition.

### Caveats

The inspected [runner source](https://github.com/vankhangfet/orca-sdlc-kit/blob/main/.orca/flow.mjs) describes launch/acceptance problems and a lower-level dispatch path involving terminal creation, worker-preamble delivery, and terminal input. Those source comments are the project's account of its experience, not independently reproduced findings here.

Its timeout and resource policies differ from ours; the source describes leaving a worker terminal open at an absolute time cap rather than establishing our required attempt-scoped stopping semantics.

This is useful implementation evidence and a source of failure cases, **not proof of independent coordinator identity, isolated cross-container access, or compatibility with our deployed Orca version**. Do not import its workarounds into the verification spike, whose approved scope is to stop and report unsupported behavior.

## Orca declarative workflow feature request

Source: [stablyai/orca issue #11229 — Declarative Workflow Builder for Deterministic Multi-Agent Pipelines](https://github.com/stablyai/orca/issues/11229)

The issue describes a gap between custom scripts sequencing Orca orchestration commands and LLM-driven routing expressed in natural language. It proposes:

- Structured YAML/JSON workflow definitions.
- A deterministic dispatcher.
- Branching based on structured worker output.
- Worker launch and completion handling.
- Human decision gates.
- Event-driven triggers and a visual workflow builder.

### Relevance and limits

Other Orca users have identified the same glue-layer concern. The issue was open when inspected. **A proposal is not evidence that the requested capability ships**, and command examples in the issue are not a version-matched runtime contract.

## OpenAI Symphony

Sources:

- [Repository and README](https://github.com/openai/symphony)
- [Specification](https://github.com/openai/symphony/blob/main/SPEC.md)

Symphony describes issue-tracker-driven, isolated coding-agent runs. Its specification separates tracker access, deterministic orchestration, workspace management, agent execution, and observability. The orchestrator owns scheduling, bounded concurrency, retries, and reconciliation.

```text
Issue tracker --> Deterministic service --> Isolated coding-agent execution
```

### Relevance

Strong architectural prior art for lifecycle ownership and responsibility boundaries. Particularly useful to compare dispatch eligibility, authoritative scheduling state, reconciliation, and failure handling.

### Differences

Its documented orchestration is polling-driven. It is not an Orca adapter or this project's Linear agent-session webhook integration. Its retry, recovery, and workspace policies must not be assumed to match ours. The README describes an engineering preview for trusted environments, not a production-readiness guarantee.

## Aurelio Semantic Router

Source: [aurelio-labs/semantic-router](https://github.com/aurelio-labs/semantic-router)

The inspected README describes:

- Embedding-based semantic routing.
- A `DecisionRouter` supporting TypeSafe's Jev and other decision models.
- Choice, yes/no, and score questions.
- A `Chain` that handles easy cases with a fast router and falls through to a slower router when uncertain.
- Calibration and confidence thresholds.

### Relevance

Direct prior art for the optional classifier-assisted routing idea:

```text
Bounded decision layer
        |
        +--> Known permitted action
        |
        +--> Escalation to a more capable decision-maker
```

### Limits

This is a decision component, not a worker lifecycle supervisor. It does not replace persistence, message correlation, permission checks, cancellation evidence, or restart recovery. Its documentation's performance and calibration claims were not independently benchmarked here.

See [Classifier-assisted orchestration](ideas/classifier-assisted-orchestration.md) for our exploratory boundaries and proposed evidence-led evaluation. No dependency or adoption decision has been made.

## Orchestrator.inc

Source: [OrchestratorInc/agent-orchestrator](https://github.com/OrchestratorInc/agent-orchestrator)

The former `ComposioHQ/agent-orchestrator` repository URL resolved to documentation identifying the project as Orchestrator.inc.

Its README describes coding-agent workers in separate branches/worktrees, routing CI and review feedback back to workers, and surfacing questions requiring human attention.

### Relevance and limits

Useful for studying feedback loops and escalation UX. It is a much broader workspace/runtime product, not a small adapter around Orca. Documented features do not establish equivalence to our runtime or lifecycle contracts.

## Naming collisions and weak matches

- [OrcaRouter-Lite](https://github.com/Continuum-AI-Corp/OrcaRouter-Lite) describes an LLM API/model router, not an Orca ADE worker supervisor. Similar names do not imply integration with our Orca runtime.
- [cultivoneuro-source/linear-orca](https://github.com/cultivoneuro-source/linear-orca) had integration-oriented repository metadata, but the inspected contents contained only a small HTML file and did not substantiate an orchestration implementation. It is not counted as verified prior art.

## Implications for this project

1. Study **Flow Kit** for Orca-specific pitfalls, not as a substitute for version-matched external-coordination proof.
2. Study **Symphony** for deterministic lifecycle boundaries and reconciliation design.
3. Study **Semantic Router** if real blocked/question cases justify evaluating Jev.
4. Keep the MVP bounded; do not expand it into a general workflow engine merely because broader tools exist.

This document changes no MVP requirements, spike budgets, safety policies, or implementation tasks. It authorizes no installation, worker launch, workaround, or runtime change.
