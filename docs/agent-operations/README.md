# Agent Operations Gateway — Community Architecture

The Agent Operations Gateway coordinates human-supervised AI agents without
exposing private business rules or production implementation.

```text
Channels / Web App / Admin Cockpit
                 |
                 v
        Agent Operations Gateway
  Session | Task | Event | Approval | Policy
                 |
      +----------+----------+
      |          |          |
      v          v          v
 Customer    Booking     Support
  Agent       Agent       Agent
      |          |          |
      +----------+----------+
                 |
                 v
     Memory, Evidence and Audit Log
```

## Public scope

This community package describes:

- persistent workspaces and sessions;
- scheduled, event-driven and condition-based tasks;
- checkpoints, validation and bounded retry;
- human approval and takeover;
- reusable skill references;
- agent identity and ownership metadata;
- runtime policy boundaries;
- structured activity and evidence records.

## Deliberately excluded

- production-ready orchestrator code;
- customer-specific integrations;
- private prompts and decision rules;
- credential handling implementation;
- commercial workflow logic;
- real tenant or customer data.

## Design principles

1. Human oversight for consequential actions.
2. Least privilege for every agent and tool.
3. Checkpoint before irreversible actions.
4. Evidence before declaring completion.
5. Memory must be reviewable and removable.
6. Policy enforcement belongs at runtime, not only in prompts.
7. Public examples must use synthetic data.
