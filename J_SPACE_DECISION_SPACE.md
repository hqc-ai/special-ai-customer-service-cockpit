# J Space / Decision Space Design

## 1. Definition

In this project, **J Space / Decision Space** means the visible, structured state of the AI’s current business decision process.

It is not a raw chain-of-thought display.

It is a practical supervision layer that helps admins understand:

- What the AI thinks the customer wants.
- What options the AI is considering.
- What information is missing.
- How confident the AI is.
- What the AI plans to do next.
- Whether human intervention is needed.

## 2. Why Not Show Raw Reasoning?

A production AI system should avoid exposing raw hidden reasoning directly.

Instead, the system should show a structured decision summary that is safe, concise, and useful for business supervision.

Bad design:

```text
Show all hidden reasoning token by token.
```

Better design:

```text
Show structured state:
- intent
- lead stage
- confidence
- next action
- risk flag
- intervention recommendation
```

## 3. Proposed J Space Fields

## 3.1 Customer Intent

What the customer appears to want.

Examples:

- Ask for price.
- Ask for location.
- Compare services.
- Request booking.
- Request legal address.
- Ask about contract.
- Ask about refund.
- Complaint.
- Unknown.

## 3.2 Lead Stage

Possible values:

- visitor
- interested
- qualified
- high_intent
- ready_to_book
- needs_human
- not_relevant

## 3.3 Service Fit

Suggested service match.

Examples:

- Virtual office.
- Business registration address.
- Private office.
- Meeting room.
- Coworking seat.
- Consulting service.
- Training course.
- Unknown.

## 3.4 Missing Information

Information the AI still needs.

Examples:

- City.
- District.
- Budget.
- Start date.
- Company type.
- Team size.
- Required invoice.
- Contract duration.

## 3.5 Confidence

A numerical or categorical confidence value.

Example:

```yaml
confidence: 0.82
confidence_label: "medium-high"
```

## 3.6 Risk Flags

Possible risk flags:

- Legal/compliance question.
- Pricing not confirmed.
- Customer is angry.
- Customer requests guarantee.
- High-value lead.
- Sensitive personal data.
- Ambiguous request.
- AI knowledge may be outdated.
- Human approval recommended.

## 3.7 Next Best Action

The action the AI should take next.

Examples:

- Ask one clarifying question.
- Recommend a specific service.
- Offer callback.
- Collect phone number.
- Ask admin to take over.
- Send booking link.
- Explain service scope.
- Escalate.

## 3.8 Admin Intervention Recommendation

Possible values:

- none
- optional
- recommended
- required

## 4. Example J Space Snapshot

```json
{
  "conversation_id": "demo-001",
  "customer_intent": "virtual office for company registration",
  "lead_stage": "high_intent",
  "service_fit": "virtual office",
  "urgency": "high",
  "missing_information": ["city", "expected registration date"],
  "confidence": 0.86,
  "risk_flags": ["legal suitability should be confirmed"],
  "next_best_action": "Ask for city and expected registration timeline, then offer admin callback.",
  "admin_intervention": "recommended"
}
```

## 5. J Space Lifecycle

```text
Initial message
      ↓
Intent detection
      ↓
Lead qualification
      ↓
Service matching
      ↓
Missing information check
      ↓
Risk and confidence evaluation
      ↓
Next-best-action selection
      ↓
Admin intervention decision
```

## 6. Human Guidance Inputs

Admins can influence J Space through structured input, such as:

```yaml
admin_guidance:
  priority_service: "virtual office"
  avoid_topic: "discount"
  ask_next: "phone number"
  escalation_required: false
```

The AI should then adjust its next action accordingly while keeping the customer conversation natural.

## 7. Governance Notes

The J Space should be:

- Visible only to authorized admins.
- Logged for review.
- Designed for supervision, not surveillance abuse.
- Separated from private hidden chain-of-thought.
- Limited to business-relevant state.
- Explainable enough for review and improvement.
