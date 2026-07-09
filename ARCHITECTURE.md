# Architecture – AI Customer Service Cockpit

## 1. Architecture Overview

The system is designed as a supervised AI customer-service layer.

```text
[Website Visitor]
        ↓
[Embedded Chat Widget]
        ↓
[Conversation Gateway]
        ↓
[AI Orchestrator]
        ↓
[Decision Space Engine]
        ↓
[Admin Cockpit]
        ↓
[Human Guidance / Approval / Takeover]
```

## 2. Main Components

## 2.1 Embedded Chat Widget

The customer-facing chat interface embedded on the website.

Responsibilities:

- Display chat messages.
- Collect customer inputs.
- Show AI replies.
- Send events to the backend.
- Support handoff to human admin if needed.

## 2.2 Conversation Gateway

The backend entry point for chat messages.

Responsibilities:

- Receive customer messages.
- Create or resume conversation sessions.
- Route messages to the AI orchestrator.
- Stream or return AI responses.
- Maintain session metadata.

## 2.3 AI Orchestrator

The main AI coordination layer.

Responsibilities:

- Interpret customer messages.
- Retrieve relevant business knowledge.
- Call service recommendation logic.
- Generate customer-facing responses.
- Update the decision-space state.
- Decide whether admin intervention is recommended.

## 2.4 Decision Space Engine

The structured reasoning-state layer.

It should not expose hidden chain-of-thought. Instead, it should expose safe, structured, business-readable state.

Example state:

```json
{
  "customer_intent": "service inquiry",
  "lead_stage": "qualified",
  "urgency": "medium",
  "recommended_action": "ask for location and timeline",
  "confidence": 0.78,
  "missing_information": ["location", "budget", "start date"],
  "risk_flags": ["pricing needs confirmation"],
  "admin_intervention": "optional"
}
```

## 2.5 Admin Cockpit

The admin-facing interface.

Suggested panels:

1. Live chat panel.
2. Decision-space panel.
3. Customer profile panel.
4. Suggested next actions.
5. Intervention controls.
6. Conversation log.
7. Lead status.

## 2.6 Human Intervention Layer

Admins can guide the AI through structured controls.

Examples:

- “Recommend service A first.”
- “Ask for phone number now.”
- “Do not mention discount yet.”
- “Escalate to admin.”
- “Take over conversation.”
- “Approve this message.”
- “Reject and rewrite.”

## 2.7 Business Knowledge Base

Stores business-specific information such as:

- Services.
- Locations.
- Pricing rules.
- Promotions.
- FAQs.
- Eligibility rules.
- Escalation rules.
- Compliance notes.

## 2.8 Audit and Trace Log

Stores important operational events:

- Customer message.
- AI reply.
- Decision-space snapshot.
- Admin intervention.
- Handoff event.
- Lead status update.
- Approval/rejection of AI action.

## 3. Data Flow

## Step 1 – Customer Message

A visitor sends a message through the website chat widget.

## Step 2 – Session Update

The conversation gateway updates the customer session and forwards the message.

## Step 3 – AI Processing

The AI orchestrator analyzes the message, retrieves business context, and prepares a reply.

## Step 4 – Decision Space Update

The decision-space engine produces a structured state update.

## Step 5 – Admin Notification

If the lead is important or uncertain, the admin cockpit receives an alert.

## Step 6 – AI Response or Approval

Depending on risk level, the AI either replies automatically or waits for admin approval.

## Step 7 – Human Intervention

The admin may guide, approve, edit, or take over the conversation.

## 4. Suggested Technical Stack

This repository does not require a fixed stack, but one possible future stack could be:

- Frontend: React / Next.js / Astro component.
- Backend: Node.js / Python.
- Realtime: WebSocket / Server-Sent Events.
- Database: PostgreSQL / SQLite for prototype.
- Vector search: pgvector / local vector database.
- AI runtime: cloud LLM or local LLM depending on deployment constraints.
- Admin auth: role-based access control.
- Logging: structured event log.

## 5. Security and Governance Considerations

Important controls should include:

- Admin-only access to decision space.
- No exposure of hidden chain-of-thought to customers.
- Logging of human intervention.
- Role-based permissions.
- Data retention rules.
- Customer data minimization.
- Safe escalation rules.
- Clear distinction between AI suggestion and human-approved decision.

## 6. Architecture Principle

The cockpit should be designed as a separate architecture package first, then integrated later into an existing website as an embedded chat module or plugin.

This reduces the risk of damaging the existing production web application while the AI workflow is still experimental.
