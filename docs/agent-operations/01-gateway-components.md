# Gateway Components

## Core components

| Component | Community responsibility |
|---|---|
| Agent Registry | Records agent ID, owner, purpose, risk tier and status |
| Session Manager | Preserves task context without exposing hidden reasoning |
| Task Router | Routes bounded work to an appropriate agent or skill |
| Scheduler | Starts approved recurring tasks |
| Event Router | Receives validated events and creates bounded tasks |
| Checkpoint Manager | Stores resumable progress and validation status |
| Approval Service | Pauses high-impact actions for human decision |
| Policy Engine | Applies least-privilege and deny-by-default rules |
| Memory Service | Stores governed facts, preferences and lessons |
| Audit Service | Records actions, evidence, approvals and outcomes |

## Important boundary

The Decision Space is a structured operational state, not a transcript of
private model reasoning. It may contain intent, assumptions, confidence,
missing information, risk flags, options and proposed next actions.
