# Orca Router

The language used to distinguish user-facing Linear interactions from Orca execution and ownership of unfinished work.

## Language

**Linear session**:
The user-facing interaction with the agent in Linear, distinct from the execution attempt performing its work. Its interaction can end while the execution attempt remains blocked.
_Avoid_: Worker session, execution attempt

**Execution attempt**:
One authoritative effort to perform a requested task through an Orca worker, distinct from its user-facing Linear session.
_Avoid_: Linear session, conversation

**Blocked attempt**:
An execution attempt awaiting clarification, not a failed, canceled, or completed attempt.
_Avoid_: Failed task, stopped worker

**Busy router**:
A router whose execution slot is occupied, while webhook handling, monitoring, and stop handling remain responsive.
_Avoid_: Unresponsive router, blocked process

**Retained worker**:
A settled worker preserved for inspection, without authority to continue its finished execution attempt.
_Avoid_: Blocked worker, running attempt

**Operator-supervised continuation**:
Work resumed directly by an operator after an attempt has blocked and its Linear interaction has ended. This continuation remains distinct from resumption through Linear.
_Avoid_: Linear reply resumption, new execution attempt
