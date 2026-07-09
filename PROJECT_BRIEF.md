# Project Brief – AI Customer Service Cockpit with J Space / Decision Space

## 1. One-Sentence Description

An AI customer-service cockpit that allows admins to observe both the live AI-customer chat and the AI’s structured decision space, then intervene when business judgment is needed.

## 2. Problem

Most AI chatbots expose only the final answer. Business owners cannot easily see:

- What the AI believes the customer wants.
- Whether the AI is pushing the correct service.
- Whether the customer is ready to buy.
- Whether the AI is uncertain.
- Whether a human should step in.
- Whether a lead is being lost.

This creates a trust gap between AI automation and real business operations.

## 3. Proposed Solution

Build an admin cockpit with two synchronized views:

### Window 1 – Live Chat

Shows the real-time conversation between the AI and the website visitor.

### Window 2 – J Space / Decision Space

Shows the AI’s current decision state, including:

- Customer intent.
- Lead stage.
- Service recommendation.
- Missing information.
- Confidence level.
- Risk flags.
- Next-best-action.
- Suggested admin intervention.

This gives the business owner a way to supervise the AI before fully delegating customer conversion tasks.

## 4. Target Users

- Service business owners.
- Sales admins.
- Customer support teams.
- Shared office operators.
- Consulting and training providers.
- SMEs that need website lead conversion.
- AI workflow designers.

## 5. Example Business Scenario

A customer visits a website and asks:

> “I need a virtual office address for company registration. Can I use it next week?”

The AI should not only answer. It should classify and expose its decision state:

```yaml
customer_intent: "Virtual office for business registration"
lead_stage: "High intent"
urgency: "High"
recommended_service: "Virtual office with business registration support"
missing_information:
  - target city
  - company type
  - required start date
risk_flags:
  - legal/address suitability must be confirmed
next_best_action: "Ask city and expected registration date, then offer admin callback"
admin_intervention_recommended: true
```

The admin can then:

- Allow the AI to continue.
- Add a private instruction.
- Take over the chat.
- Approve a quote or booking step.

## 6. Expected Benefits

- Better control over AI sales conversations.
- Higher lead conversion.
- Reduced response time.
- More consistent service recommendations.
- Earlier human intervention for important leads.
- Better traceability of AI decisions.
- Safer AI deployment in customer-facing workflows.

## 7. Scope

### In Scope

- Customer-service AI cockpit concept.
- Admin observation workflow.
- Decision-space state design.
- Human intervention mechanism.
- Lead qualification and service recommendation logic.
- Architecture for later integration into web applications.

### Out of Scope for Current Phase

- Full production implementation.
- Payment processing.
- Legal decision automation.
- Fully autonomous contract approval.
- Replacement of human sales/admin teams.

## 8. Success Criteria

A future implementation should be considered successful if it can:

- Handle simple customer conversations.
- Detect high-intent leads.
- Show decision-space state clearly to admins.
- Allow admins to intervene safely.
- Log important AI decisions.
- Improve lead handling without losing human control.

## 9. Future Direction

The long-term direction is to develop this into a reusable AI cockpit layer that can be embedded into websites as a plugin or module.
