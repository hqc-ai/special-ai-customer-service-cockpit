# Long-Running Task Lifecycle

```text
Requested
   |
   v
Validated --> Rejected
   |
   v
Planned
   |
   v
Running <--> Checkpointed
   |
   +--> Awaiting Approval
   |
   +--> Retrying (bounded)
   |
   v
Verifying
   |
   +--> Failed
   |
   v
Completed
```

## Minimum controls

- clear task owner and agent owner;
- defined success criteria;
- time, cost and tool budget;
- maximum retry count;
- checkpoint interval;
- approval rules;
- evidence requirements;
- cancellation and timeout behavior;
- final human-readable handoff.

A successful API response or build is not sufficient proof that a task achieved
its intended business outcome.
