# Roadmap

## Phase 0 – Concept Definition

Status: current phase.

Goals:

- Define the business problem.
- Describe the AI cockpit concept.
- Separate this Special AI phase from the existing production web application.
- Document the decision-space idea.
- Prepare public GitHub introduction files.

Deliverables:

- README.
- Project brief.
- Architecture overview.
- J Space / decision-space design.
- Roadmap.
- Author profile.

## Phase 1 – Prototype Design

Goals:

- Create basic UI wireframe.
- Define data schema for conversation state.
- Define J Space JSON format.
- Define admin intervention actions.
- Prepare sample conversations.

Possible deliverables:

- `/examples/conversation-samples.md`
- `/schemas/decision-space.schema.json`
- `/wireframes/admin-cockpit.md`

## Phase 2 – Local Prototype

Goals:

- Build a local proof of concept.
- Simulate website chat.
- Simulate AI state update.
- Display live chat and J Space side by side.
- Allow admin guidance input.

Possible stack:

- React or simple web frontend.
- Node.js or Python backend.
- Local database.
- Local or cloud LLM.

## Phase 3 – Business Knowledge Integration

Goals:

- Connect service catalog.
- Add location data.
- Add FAQ knowledge.
- Add lead qualification rules.
- Add escalation rules.

Example business domains:

- Virtual office.
- Company registration address.
- Private office.
- Meeting room.
- Training and consulting service.

## Phase 4 – Human-in-the-Loop Workflow

Goals:

- Add admin notification.
- Add approval before sensitive replies.
- Add takeover mode.
- Add lead tagging.
- Add conversation review.

## Phase 5 – Website Plugin Integration

Goals:

- Package the customer chat widget.
- Embed into a website.
- Keep admin cockpit separate.
- Avoid disrupting the production web application.

## Phase 6 – Governance and Auditability

Goals:

- Add trace logs.
- Add decision snapshots.
- Add admin action history.
- Add privacy and data-retention rules.
- Add compliance review checklist.

Relevant governance directions:

- AI transparency.
- Human oversight.
- Data minimization.
- Security controls.
- Management system alignment.

## Phase 7 – Production Pilot

Goals:

- Test with real but limited business scenarios.
- Start with human-supervised mode.
- Track lead conversion.
- Review errors and missed opportunities.
- Improve business rules and AI behavior.

## Phase 8 – Controlled Automation

Goals:

- Automate low-risk cases.
- Keep human approval for high-impact cases.
- Build service-specific playbooks.
- Measure performance and customer satisfaction.
