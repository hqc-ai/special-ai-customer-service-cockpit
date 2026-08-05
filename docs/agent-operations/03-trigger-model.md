# Trigger Model

The gateway supports three public architectural patterns.

## Scheduled

Runs on an approved timetable, such as preparing a daily unresolved-lead list.

## Event-driven

Runs after a validated business event, such as a booking request or new support case.

## Condition-based

Checks whether an explicit condition is true, such as evidence nearing expiry.
A condition check must not generate repeated notifications when no action is needed.

## Safety rules

- Every trigger identifies its owner.
- Inputs are validated before task creation.
- Duplicate events are idempotently handled.
- High-impact actions require approval.
- Trigger frequency is bounded.
- Failure does not silently create an endless retry loop.
