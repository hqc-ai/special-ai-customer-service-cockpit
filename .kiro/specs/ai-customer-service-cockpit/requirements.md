# Requirements Document

## Introduction

The AI Customer Service Cockpit is a standalone, embeddable web chat system designed for 5SOffice. It enables an AI agent to handle real-time conversations with website visitors about virtual offices, coworking spaces, meeting rooms, and business address services. A human admin can observe every conversation through a structured cockpit interface, view the AI's decision state (J Space), and intervene at multiple levels — from silent observation to full takeover.

The system is architected as a separate service to allow future embedding into the 5S Webapp via a JavaScript widget, without tight coupling to existing infrastructure. Human-in-the-loop control and strict business guardrails are first-class requirements, not optional additions.

---

## Glossary

- **Visitor**: An anonymous or identified website visitor who initiates a chat session.
- **AI_Agent**: The automated assistant that responds to Visitor messages based on approved knowledge and business rules.
- **Admin**: A human operator (sales staff or manager) who monitors, guides, and optionally takes over conversations.
- **System_Owner**: The operator who configures business rules, knowledge content, pricing boundaries, and escalation policies.
- **Conversation**: A single chat session between a Visitor and the AI_Agent, optionally involving an Admin.
- **Message**: A single unit of text exchanged within a Conversation, attributed to Visitor, AI_Agent, or Admin.
- **Decision_State**: The structured internal state object (J Space) that the AI_Agent maintains per Conversation, representing intent, lead stage, known facts, risk flags, and recommended actions.
- **J_Space**: Synonym for Decision_State. The "decision space" panel shown to Admins in the cockpit.
- **Intervention_Level**: One of five escalating modes of Admin involvement in a Conversation (Silent Observation → Guidance → Constraint → Approval Required → Takeover).
- **Approval_Request**: A pending proposed AI_Agent response that requires Admin review before being sent to the Visitor.
- **Admin_Note**: An internal instruction from Admin to AI_Agent, not visible to the Visitor.
- **Lead**: A Visitor record enriched with contact information and service interest, created or updated during a Conversation.
- **Knowledge_Base**: The repository of approved content (FAQs, service descriptions, pricing policy, escalation rules) used by AI_Agent.
- **Guardrail**: A business rule enforced by the system that prevents the AI_Agent from making unauthorized commitments.
- **Audit_Log**: An immutable record of all significant events in a Conversation.
- **Chat_Widget**: The embeddable frontend component rendered on the 5SOffice website.
- **Cockpit**: The Admin-facing dashboard showing active conversations, J Space, and intervention controls.
- **Orchestrator**: The backend service that manages AI_Agent behavior, applies business rules, and routes messages.
- **Notification_Service**: The component that alerts Admins of new conversations or events requiring attention.
- **Realtime_Layer**: The infrastructure (WebSocket or equivalent) that delivers messages and state updates with low latency.

---

## Requirements

### Requirement 1: Web Chat Widget

**User Story:** As a Visitor, I want to open a chat window on the website so that I can ask questions about 5SOffice services and receive real-time answers.

#### Acceptance Criteria

1. THE Chat_Widget SHALL render as an embeddable component activated by including a single script tag on any webpage.
2. WHEN a Visitor opens the Chat_Widget, THE Chat_Widget SHALL create a new Conversation and connect to the Realtime_Layer within 3 seconds; IF the connection cannot be established within 3 seconds, THE Chat_Widget SHALL display an error message and provide a retry option without requiring a page reload.
3. THE Chat_Widget SHALL operate without requiring Visitor authentication or prior account creation.
4. WHEN a Visitor submits a message, THE Chat_Widget SHALL display the message within 500 milliseconds and show a typing indicator while awaiting the AI_Agent response; WHEN the AI_Agent response is received, THE Chat_Widget SHALL remove the typing indicator and display the response.
5. THE Chat_Widget SHALL collect Visitor contact information (name, phone number, email address, company name, and service interest) through structured prompts or natural conversation flow; WHEN collecting phone number or email address, THE Chat_Widget SHALL validate the format before accepting the input.
6. THE Chat_Widget SHALL display the full message history of the current Conversation for the duration of the session.
7. THE Chat_Widget SHALL render correctly on desktop viewports of 1024px width or greater and on mobile viewports of 768px width or less, adapting layout to the available screen size.
8. WHEN an Admin takes over a Conversation, THE Chat_Widget SHALL continue to display messages without indicating to the Visitor whether responses originate from AI_Agent or Admin.
9. WHEN a Conversation is definitively closed, THE Chat_Widget SHALL display a closure message indicating the conversation has ended and immediately disable the message input field in the same operation.
10. WHERE the host page uses HTTPS, THE Chat_Widget SHALL transmit all data exclusively over encrypted connections.

---

### Requirement 2: Realtime Messaging

**User Story:** As a Visitor and as an Admin, I want all messages and state updates to appear instantly so that the conversation feels live and the Admin can react without delay.

#### Acceptance Criteria

1. THE Realtime_Layer SHALL deliver messages from Visitor to AI_Agent and from AI_Agent to Visitor with end-to-end latency not exceeding 2 seconds under normal load, where normal load is defined as up to 50 simultaneous active Visitor sessions.
2. WHEN the AI_Agent updates the Decision_State, THE Realtime_Layer SHALL deliver the Decision_State update to the Admin Cockpit within 1 second; IF delivery fails, THE Realtime_Layer SHALL retry up to 3 times within 5 seconds and, upon exhaustion of retries, SHALL indicate a stale state in the Cockpit J Space panel.
3. WHEN the Realtime_Layer connection is interrupted, THE Chat_Widget SHALL attempt reconnection automatically using exponential backoff starting at 1 second and doubling each attempt up to a maximum interval of 30 seconds, with a maximum of 10 reconnection attempts.
4. WHEN reconnection succeeds after an interruption, THE Chat_Widget SHALL retrieve and display all missed messages in chronological order before accepting any new messages from the Visitor.
5. THE Realtime_Layer SHALL support simultaneous connections from multiple Admin sessions observing the same Conversation without message duplication.
6. WHEN a Visitor sends a message, THE Realtime_Layer SHALL route the message to the Orchestrator regardless of whether routing to active Admin sessions succeeds; failure to deliver to one or more Admin sessions SHALL not prevent the message from reaching the Orchestrator.
7. IF the Chat_Widget exhausts all reconnection attempts without success, THEN THE Chat_Widget SHALL stop retrying, display a connection error message to the Visitor, and provide a manual retry button.

---

### Requirement 3: AI Agent Orchestration

**User Story:** As a Visitor, I want the AI Agent to understand my questions and give accurate, policy-compliant answers so that I get useful information without being misled.

#### Acceptance Criteria

1. WHEN a Visitor message is received, THE Orchestrator SHALL classify the intent of the message and update the Decision_State before generating a response.
2. THE Orchestrator SHALL generate responses using only content retrieved from the Knowledge_Base and the current Decision_State context.
3. WHEN the Orchestrator generates a response, THE Orchestrator SHALL evaluate the response against all active Guardrails before sending it to the Visitor.
4. IF a response violates a Guardrail, THEN THE Orchestrator SHALL withhold the response, prevent it from being delivered to the Visitor under any circumstance, mark the action as restricted in the Decision_State, update the Decision_State with a risk flag, and send the Visitor a message indicating the topic cannot be addressed directly and offering escalation to an Admin.
5. WHEN an Admin_Note is added to a Conversation, THE Orchestrator SHALL incorporate the Admin_Note into the AI_Agent's context for all subsequent responses in that Conversation.
6. THE Orchestrator SHALL maintain the full conversation context across all turns within a single Conversation session, limited to the current Conversation session only and not persisted across separate Conversations.
7. WHEN the AI_Agent determines that a proposed response requires admin approval, THE Orchestrator SHALL create an Approval_Request and set the Conversation status to "Waiting for Approval" rather than sending the response.
8. WHILE a Conversation is in "Admin Takeover" status, THE Orchestrator SHALL suspend direct AI_Agent responses to the Visitor and limit the AI_Agent to providing private assistance to the Admin.
9. THE Orchestrator SHALL make private AI_Agent assistance available to the Admin during any Conversation status upon Admin request, not only during Admin Takeover.
10. THE Orchestrator SHALL update the Decision_State after every Visitor message, AI_Agent response, and Admin_Note.
11. WHEN a Conversation is closed, THE Orchestrator SHALL produce a structured conversation summary containing: final intent classification, final lead_stage, list of service interests identified, list of known facts collected, and a list of all Admin interventions that occurred during the Conversation.

---

### Requirement 4: Decision State (J Space) Engine

**User Story:** As an Admin, I want to see the AI Agent's structured internal state so that I can understand the AI's reasoning and intervene with full context without reading raw AI chain-of-thought.

#### Acceptance Criteria

1. THE Decision_State SHALL contain the following fields for every active Conversation: conversation_id, intent, lead_stage, service_interest, known_facts, missing_information, risk_flags, recommended_next_action, allowed_actions, restricted_actions, requires_admin_approval, approval_reason, confidence_level (a numeric value between 0.0 and 1.0 inclusive), admin_notes, and last_updated_at.
2. WHEN the AI_Agent updates the Decision_State, THE Decision_State Engine SHALL persist the updated state to the database; IF persistence fails, THE Decision_State Engine SHALL return an error and SHALL NOT push the update to Admin sessions; WHEN persistence succeeds, THE Decision_State Engine SHALL push the update to all Admin sessions observing that Conversation.
3. THE Decision_State SHALL represent structured business-level classifications and SHALL NOT expose raw AI chain-of-thought or internal model reasoning text.
4. WHEN a Guardrail is triggered, THE Decision_State Engine SHALL record the triggered Guardrail identifier and the associated risk flag in the Decision_State.
5. THE Decision_State Engine SHALL maintain a versioned history of all Decision_State updates for a Conversation, retaining the full history for the duration of the Conversation and for a minimum of 12 months thereafter for Audit_Log purposes.
6. WHEN the AI_Agent changes the lead_stage field, THE Decision_State Engine SHALL emit an event that triggers a Lead record update; IF the Lead record update fails, THE Decision_State Engine SHALL log the failure in the Audit_Log and retry up to 3 times before marking the event as failed.
7. FOR ALL valid Conversation sessions, serializing and deserializing the Decision_State SHALL produce a Decision_State object where every field is equal in type and value to the original, with no fields omitted or defaulted.

---

### Requirement 5: Admin Cockpit Dashboard

**User Story:** As an Admin, I want a unified dashboard that shows all active conversations, live chat, and decision states so that I can monitor and manage AI performance across all concurrent sessions.

#### Acceptance Criteria

1. THE Cockpit SHALL display a list of all active Conversations with their current status, Visitor identity (if collected), and elapsed time.
2. WHEN a new Conversation is created, THE Cockpit SHALL add it to the active conversation list in real time without requiring a page refresh.
3. WHEN an Admin selects a Conversation, THE Cockpit SHALL display a two-panel view: Window A (live chat transcript) and Window B (J Space / Decision_State view); if one panel fails to load, THE Cockpit SHALL display the successfully loaded panel and indicate the loading failure for the other.
4. THE Cockpit SHALL display the current Conversation status using one of the following labels: "AI Handling", "Waiting for Approval", "Admin Observing", "Admin Guided", "Admin Takeover", "Closed", "Converted to Lead", or "Follow-up Needed".
5. WHEN the Decision_State is updated, THE Cockpit SHALL refresh the J Space panel automatically without requiring Admin interaction.
6. THE Cockpit SHALL provide an Admin_Note input panel that allows the Admin to submit internal instructions to the AI_Agent at any time during an active Conversation.
7. THE Cockpit SHALL provide an Approval Panel that displays all pending Approval_Requests with the AI_Agent's proposed response and options to approve, edit, or reject.
8. THE Cockpit SHALL provide a Takeover button that transitions the Conversation to "Admin Takeover" status when activated by the Admin.
9. THE Cockpit SHALL display the Lead information panel showing all collected Visitor contact details and service interest for the selected Conversation.
10. THE Cockpit SHALL display the Audit_Log for the selected Conversation, showing all significant events in chronological order.
11. WHERE the Admin lacks RBAC permission to view contact data, THE Cockpit SHALL hide all Visitor contact fields — including name, phone number, email address, and company name — treating contact information as a single permission scope.

---

### Requirement 6: Admin Intervention Levels

**User Story:** As an Admin, I want graduated intervention controls so that I can respond proportionally — from passively observing to fully taking over — without disrupting the Visitor experience unnecessarily.

#### Acceptance Criteria

1. THE Cockpit SHALL support five Intervention_Levels per Conversation: Level 1 (Silent Observation), Level 2 (Guidance), Level 3 (Constraint), Level 4 (Approval Required), and Level 5 (Admin Takeover); WHERE multiple Admins are active on the same Conversation at different levels, THE Orchestrator SHALL apply the highest active Intervention_Level.
2. WHEN an Admin opens a Conversation in the Cockpit without taking any action, THE Cockpit SHALL record the Admin as a silent observer at Intervention_Level 1 without altering AI_Agent behavior.
3. WHEN an Admin submits an Admin_Note, that submission SHALL activate Intervention_Level 2 for that Admin; THE Orchestrator SHALL incorporate the note into AI_Agent context and update the Conversation status to "Admin Guided"; IF the Conversation is already at a level higher than 2, the level SHALL remain unchanged and the note SHALL still be incorporated.
4. WHEN an Admin applies a Decision_Constraint at Intervention_Level 3, THE Orchestrator SHALL add the specified action to the restricted_actions list of the Decision_State for the remainder of the Conversation; WHEN Intervention_Level 5 is activated, active Decision_Constraints SHALL be suspended but not removed, and SHALL be restored when control returns to the AI_Agent.
5. WHEN an Admin sets Intervention_Level 4 on a Conversation, THE Orchestrator SHALL require Admin approval for all subsequent AI_Agent responses before they are sent to the Visitor, including any response already in-flight at the moment Level 4 is activated.
6. WHEN an Admin activates Takeover at Intervention_Level 5, THE Orchestrator SHALL suspend AI_Agent responses to the Visitor, route all outgoing messages from the Admin directly to the Visitor, and suspend all active Decision_Constraints so that the Admin's direct messages are not subject to those restrictions.
7. WHILE a Conversation is in Intervention_Level 5, THE Orchestrator SHALL continue to provide private AI_Agent assistance to the Admin upon request.
8. WHEN an Admin initiates return of control to the AI_Agent after a Takeover, THE Cockpit SHALL present a confirmation step; WHEN the Admin confirms, THE Orchestrator SHALL restore any previously suspended Decision_Constraints, resume AI_Agent responses, and update the Conversation status to "AI Handling".
9. WHEN any Intervention_Level is activated or deactivated, THE Audit_Log SHALL record the action, the Admin's identity, and the timestamp.
10. WHEN an Admin changes from any Intervention_Level to any other Intervention_Level (including skipping levels or downgrading), THE change SHALL be permitted without restriction; WHEN downgrading from a higher to a lower level, THE Audit_Log SHALL record the downgrade action, the Admin's identity, and the timestamp.

---

### Requirement 7: Approval Workflow

**User Story:** As an Admin, I want to review and approve sensitive AI responses before they reach the Visitor so that no unauthorized commitments — such as price quotes, booking confirmations, or legal statements — are made without human sign-off.

#### Acceptance Criteria

1. THE Orchestrator SHALL treat the following action categories as approval-gated: final price quotation, room availability confirmation, booking confirmation, deposit or payment request, legal or registration suitability assessment, contract commitment, special discount offer, and refund or complaint resolution.
2. WHEN the AI_Agent generates a response in an approval-gated category, THE Orchestrator SHALL create an Approval_Request containing the proposed response text, the action category, and the current Decision_State snapshot.
3. WHEN an Approval_Request is created, THE Notification_Service SHALL alert all active Admin sessions for that Conversation within 10 seconds.
4. WHEN an Admin approves an Approval_Request without edits and the response is successfully delivered to the Visitor, THE Orchestrator SHALL log the approval event in the Audit_Log; IF response delivery fails, THE Orchestrator SHALL not log the approval and SHALL retry delivery up to 3 times at 10-second intervals before logging a delivery failure event.
5. WHEN an Admin edits and approves an Approval_Request and the edited response is successfully delivered to the Visitor, THE Orchestrator SHALL log the original proposed text, the final sent text, and the approval event in the Audit_Log; IF response delivery fails, THE Orchestrator SHALL not log the approval and SHALL retry delivery up to 3 times at 10-second intervals before logging a delivery failure event.
6. WHEN an Admin rejects an Approval_Request, THE Orchestrator SHALL discard the proposed response, notify the AI_Agent of the rejection reason, and log the rejection event and rejection reason in the Audit_Log separately from approval events.
7. WHEN an Admin activates Takeover from the Approval Panel, THE Orchestrator SHALL transition the Conversation to Intervention_Level 5, cancel all pending Approval_Requests, and notify the AI_Agent that the pending requests have been cancelled.
8. IF an Approval_Request remains unreviewed for more than 5 minutes, THEN THE Notification_Service SHALL send a second escalation alert to the Admin.
9. IF an Approval_Request remains unreviewed for more than 15 minutes total, THEN THE Orchestrator SHALL automatically reject the Approval_Request, notify the AI_Agent of the auto-rejection, and log the auto-rejection event and reason ("timeout") in the Audit_Log.

---

### Requirement 8: Knowledge Base

**User Story:** As the System_Owner, I want the AI Agent to answer only from approved content so that visitors receive accurate, policy-compliant information about 5SOffice services.

#### Acceptance Criteria

1. THE Knowledge_Base SHALL store approved content for the following categories: service descriptions, frequently asked questions, pricing policy, contract and invoice policy, deposit and booking policy, escalation rules, and legal disclaimers.
2. WHEN the AI_Agent retrieves content, THE Knowledge_Base SHALL return only content with an "approved" status.
3. THE Knowledge_Base SHALL support content retrieval by semantic similarity search for natural-language queries, using a System_Owner-configurable similarity threshold with a default value of 0.70 on a 0.0–1.0 scale.
4. WHEN the System_Owner publishes a content update, THE Knowledge_Base SHALL make the updated content available to the AI_Agent within 60 seconds under normal operating conditions; IF the update cannot be propagated within 60 seconds, THE Knowledge_Base SHALL retry up to 3 times within a 10-minute window and set the content item status to "pending" until propagation succeeds; WHEN propagation fails after all retries, THE Knowledge_Base SHALL log the failure and notify the System_Owner.
5. WHEN the System_Owner marks a content item as "draft" or "archived", THE Knowledge_Base SHALL exclude that item from all AI_Agent retrieval operations.
6. THE Knowledge_Base SHALL record the author, creation date, last modified date, and approval status for every content item.
7. IF the Knowledge_Base returns no approved content with a similarity score at or above the configured threshold for a Visitor query, THEN THE AI_Agent SHALL send the Visitor a message indicating that no information is available for that topic and offer to escalate to an Admin.
8. IF the Knowledge_Base returns approved content with a similarity score at or above the configured threshold but below 0.90, THEN THE AI_Agent SHALL provide the available approved answer and additionally offer escalation to an Admin.

---

### Requirement 9: Business Guardrails

**User Story:** As the System_Owner, I want strict business rules enforced automatically so that the AI Agent never makes promises the company cannot keep or actions that require human authority.

#### Acceptance Criteria

1. THE Orchestrator SHALL enforce Guardrail GUARDRAIL-REG: the AI_Agent SHALL NOT guarantee company registration success in any response; IF a violation is detected, THE Orchestrator SHALL withhold the response, add GUARDRAIL-REG to risk_flags in the Decision_State, mark the action as restricted, and log the event in the Audit_Log.
2. THE Orchestrator SHALL enforce Guardrail GUARDRAIL-LEGAL: the AI_Agent SHALL NOT provide legal advice or legal suitability assessments in any response; IF a violation is detected, THE Orchestrator SHALL withhold the response, add GUARDRAIL-LEGAL to risk_flags, mark the action as restricted, and log the event in the Audit_Log.
3. THE Orchestrator SHALL enforce Guardrail GUARDRAIL-AVAILABILITY: the AI_Agent SHALL NOT confirm room or office availability without a verified inventory record or explicit Admin approval; IF a violation is detected, THE Orchestrator SHALL withhold the response, add GUARDRAIL-AVAILABILITY to risk_flags, mark the action as restricted, and log the event in the Audit_Log.
4. THE Orchestrator SHALL enforce Guardrail GUARDRAIL-PRICING: the AI_Agent SHALL NOT quote a final price without an approved pricing source from the Knowledge_Base; IF a violation is detected, THE Orchestrator SHALL withhold the response, add GUARDRAIL-PRICING to risk_flags, mark the action as restricted, and log the event in the Audit_Log.
5. THE Orchestrator SHALL enforce Guardrail GUARDRAIL-PAYMENT: the AI_Agent SHALL NOT request or process deposit or payment information without explicit Admin approval; IF a violation is detected, THE Orchestrator SHALL withhold the response, add GUARDRAIL-PAYMENT to risk_flags, mark the action as restricted, and log the event in the Audit_Log.
6. THE Orchestrator SHALL enforce Guardrail GUARDRAIL-FABRICATION: the AI_Agent SHALL NOT invent or fabricate office locations, room capacities, or service specifications not present in the Knowledge_Base; IF a violation is detected, THE Orchestrator SHALL withhold the response, add GUARDRAIL-FABRICATION to risk_flags, mark the action as restricted, and log the event in the Audit_Log.
7. WHEN a Visitor message contains any of the following sensitive topics — legal dispute, formal complaint, request for medical advice, or request for financial advice — THE Orchestrator SHALL withhold any AI_Agent response to that topic, create an Approval_Request, add the applicable guardrail identifier to risk_flags in the Decision_State, and alert active Admin sessions via the Notification_Service.
8. WHEN any Guardrail (GUARDRAIL-REG, GUARDRAIL-LEGAL, GUARDRAIL-AVAILABILITY, GUARDRAIL-PRICING, GUARDRAIL-PAYMENT, or GUARDRAIL-FABRICATION) is enforced, THE Audit_Log SHALL record the specific Guardrail identifier, the AI_Agent's original proposed action text, and the UTC timestamp.

---

### Requirement 10: Lead Management

**User Story:** As an Admin and as the System_Owner, I want the system to track and qualify leads from conversations so that the sales team can follow up effectively after each interaction.

#### Acceptance Criteria

1. IF a Visitor provides at least one of the following — name, phone number, or email address — THEN THE Orchestrator SHALL create a Lead record associated with the current Conversation.
2. WHEN the Decision_State lead_stage field is updated, THE Orchestrator SHALL update the corresponding Lead record with the new lead_stage value; valid lead_stage values are: "New", "Qualified", "Proposal", "Converted", and "Closed".
3. THE Lead record SHALL store: visitor identifier, name, phone number, email address, company name, service interest list, lead_stage, associated Conversation identifier, and last activity timestamp.
4. WHEN a Visitor provides additional contact information (name, phone number, email address, or company name) during a Conversation where a Lead record already exists, THE Orchestrator SHALL update the existing Lead record with the new information without creating a duplicate record.
5. WHEN a Conversation is closed, THE Cockpit SHALL prompt the Admin to review and confirm the Lead status; IF the Admin does not take action within 5 minutes of the prompt, THE System SHALL finalize the Conversation record with the Lead status set to the last known lead_stage value.
6. THE Cockpit SHALL allow the Admin to set the Lead status to one of: "Active", "Converted", "Follow-up Needed", "Unqualified", or "Closed — No Interest".
7. THE Lead record SHALL be queryable by System_Owner users and exportable in CSV format, filterable by lead_stage, date range, and service interest, for CRM handoff.

---

### Requirement 11: Notification Service

**User Story:** As an Admin, I want to be notified immediately when a new conversation starts or when my attention is required so that I can respond before the Visitor loses patience.

#### Acceptance Criteria

1. WHEN a new Conversation is created, THE Notification_Service SHALL deliver an alert to all online Admin sessions within 5 seconds.
2. WHEN an Approval_Request is created, THE Notification_Service SHALL deliver an alert to all active Admin sessions for that Conversation within 10 seconds.
3. THE Notification_Service SHALL support in-app browser push notifications as the primary delivery channel for the MVP.
4. WHEN an Admin has not acknowledged a new Conversation notification within 2 minutes of the notification being delivered, THE Notification_Service SHALL repeat the alert.
5. THE Notification_Service SHALL record every notification event, its delivery status, and the Admin's acknowledgment time in the Audit_Log.
6. WHERE the Admin has granted browser notification permission, THE Notification_Service SHALL deliver notifications even when the Admin's browser tab is not in focus.

---

### Requirement 12: Audit Log

**User Story:** As the System_Owner, I want a complete, immutable log of all conversation events so that I can review AI behavior, audit Admin interventions, and investigate any disputed interactions.

#### Acceptance Criteria

1. THE Audit_Log SHALL record the following event types: Visitor message received, AI_Agent response sent, Decision_State updated, Admin_Note added, Decision_Constraint applied, Approval_Request created, Approval_Request approved, Approval_Request edited and approved, Approval_Request rejected, Admin Takeover activated, Admin returned control to AI_Agent, Lead status updated, and Conversation closed.
2. THE Audit_Log SHALL be append-only; no event record SHALL be modified or deleted after creation.
3. EACH Audit_Log entry SHALL contain: event type, actor (Visitor, AI_Agent, or Admin identifier), Conversation identifier, event payload, and UTC timestamp.
4. THE Audit_Log SHALL be queryable by Conversation identifier, actor, event type, and time range.
5. WHEN an Audit_Log entry is written, THE Audit_Log SHALL confirm persistence before the originating operation is considered complete.
6. THE Audit_Log SHALL retain all records for a minimum of 12 months.

---

### Requirement 13: Security and Access Control

**User Story:** As the System_Owner, I want role-based access control and secure data handling so that Visitor data is protected and Admin capabilities are appropriately restricted.

#### Acceptance Criteria

1. THE Cockpit SHALL require Admin authentication before granting access to any conversation data or intervention controls.
2. THE System SHALL define RBAC roles, where each role may have a different subset of the following permissions: view conversations, add Admin_Notes, apply Decision_Constraints, approve Approval_Requests, activate Takeover, manage Knowledge_Base, and configure system settings.
3. THE Orchestrator SHALL enforce that Admin_Notes and internal Decision_State data are never included in messages sent to the Visitor.
4. THE System SHALL transmit all data between components over TLS-encrypted connections.
5. THE System SHALL apply rate limiting to the Chat_Widget endpoint to prevent abuse, rejecting requests that exceed 30 messages per Visitor session per minute.
6. IF a Visitor submits input that contains HTML, script tags, or SQL injection patterns, THEN THE System SHALL sanitize the input before processing or storing it.
7. THE System SHALL store Visitor contact information (phone number, email address) using encryption at rest.
8. THE Cockpit SHALL log all Admin login events, including timestamp, IP address, and success or failure status, in the Audit_Log.

---

### Requirement 14: Widget Embeddability and Future Integration

**User Story:** As the System_Owner, I want the chat widget to be embeddable on any webpage via a single script tag so that it can be added to the 5S Webapp and other sites in the future without modifying the host application.

#### Acceptance Criteria

1. THE Chat_Widget SHALL be activatable on any HTML page by including a single script element with a `data-site` attribute identifying the deployment target.
2. THE Chat_Widget SHALL not conflict with or modify the host page's global JavaScript namespace, CSS styles, or DOM structure outside its own container.
3. THE Chat_Widget SHALL accept configuration parameters via `data-*` HTML attributes on the script element, including at minimum: site identifier, primary color, and widget position.
4. THE System SHALL expose a documented REST API that allows future integration with external systems such as CRM platforms, room availability services, and pricing engines.
5. THE System SHALL be deployable as an independent service with no runtime dependency on the 5S Webapp codebase or infrastructure, from initial deployment onward.

---

### Requirement 15: Conversation State Lifecycle

**User Story:** As an Admin and as the System_Owner, I want conversations to follow a defined lifecycle with predictable transitions so that the system behaves consistently and the Audit_Log reflects accurate state history.

#### Acceptance Criteria

1. WHEN a Visitor initiates a chat, THE System SHALL create a Conversation in "AI Handling" status.
2. THE Conversation status SHALL transition only through the following defined paths:
   - "AI Handling" → "Waiting for Approval" (Guardrail triggers Approval_Request)
   - "AI Handling" → "Admin Observing" (Admin opens the Conversation)
   - "AI Handling" → "Admin Takeover" (Admin activates Takeover)
   - "Admin Observing" → "Admin Guided" (Admin submits Admin_Note)
   - "Admin Observing" → "Admin Takeover" (Admin activates Takeover)
   - "Admin Guided" → "Admin Takeover" (Admin activates Takeover)
   - "Waiting for Approval" → "AI Handling" (Admin approves or rejects)
   - "Waiting for Approval" → "Admin Takeover" (Admin activates Takeover)
   - Any active status → "Closed" (Admin or system closes Conversation)
   - "Closed" → "Converted to Lead" or "Follow-up Needed" (Admin updates Lead status)
3. WHEN a Conversation status changes, THE Audit_Log SHALL record the previous status, the new status, the actor, and the timestamp; IF the Audit_Log write fails, THE System SHALL roll back the status change and return an error rather than allowing the status change to persist without an audit record.
4. IF a Visitor session disconnects without the Conversation being explicitly closed, THEN THE System SHALL mark the Conversation as "Closed" after a configurable inactivity timeout (default: 30 minutes).
5. THE System SHALL prevent invalid Conversation status transitions; any attempt to perform an action that would result in an undefined transition SHALL be rejected with an error logged in the Audit_Log.
