# Runtime Policy Pattern

The effective permission is the intersection of all applicable policies.

```text
Enterprise ceiling
      ∩
Workspace policy
      ∩
Agent policy
      ∩
Task policy
      =
Effective permission
```

A lower layer may reduce permission but must not exceed an upper-layer ceiling.

## Example controls

- allowlisted tools;
- denied commands and paths;
- data classification constraints;
- human approval thresholds;
- maximum task duration and cost;
- network destination allowlist;
- sandbox requirement;
- output redaction;
- immutable or append-only activity records.
