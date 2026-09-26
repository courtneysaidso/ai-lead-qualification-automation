# AI-Assisted Lead Qualification & Lifecycle Automation

An end-to-end lead qualification workflow that combines deterministic business rules with structured LLM analysis to evaluate inbound opportunities, calculate qualification scores, route leads by priority, synchronize CRM records, and trigger appropriate internal and external follow-up.

Built with **n8n, OpenAI, HubSpot, Slack, webhooks, and SMTP email**.

> **Portfolio implementation artifact:** This repository contains a sanitized export of a workflow developed as an independent portfolio project. It is intended to demonstrate system architecture, workflow logic, AI integration, validation, routing, and implementation decisions rather than serve as a plug-and-play template.

---

## Overview

Inbound business leads often contain a mix of structured and unstructured information.

Some qualification criteria are objective and can be evaluated consistently using predefined rules:

- Budget
- Timeline
- Project readiness

Other criteria require interpretation:

- How closely does the prospect's problem align with available services?
- How clearly has the prospect defined the problem and desired outcome?
- What appears to be the primary operational need?
- What areas should be explored during discovery?

Rather than asking an LLM to make the entire qualification decision, this workflow separates deterministic business logic from probabilistic AI analysis.

Both evaluations run independently, are merged into a final qualification score, and then drive downstream routing.

---

## Architecture

```text
Inbound Webhook
      │
      ▼
Normalize Intake Data
      │
      ├──────────────► Deterministic Business Rules
      │
      └──────────────► Structured LLM Analysis
                              │
              ┌───────────────┘
              ▼
         Merge Results
              │
              ▼
     Validate + Final Score
              │
              ▼
      Qualification Routing
        │      │      │      │
       High  Standard Review  Low
        │      │      │      │
        └──── CRM / Alerts / Follow-up
```

The intake payload is normalized before qualification so downstream logic operates on a consistent internal data structure rather than depending directly on the original form payload.

From there, deterministic business rules and LLM-based qualitative analysis execute independently before their results are merged.

---

## Qualification Model

The workflow calculates a **100-point qualification score**.

### Deterministic Business Rules — 55 points

| Dimension         | Maximum |
| ----------------- | ------: |
| Budget            |      25 |
| Timeline          |      20 |
| Project readiness |      10 |
| **Total**         |  **55** |

These dimensions are calculated directly from predefined business rules.

Keeping objective qualification criteria outside the LLM makes those decisions predictable, inspectable, and easy to modify as business requirements change.

### Structured LLM Analysis — 45 points

| Dimension       | Maximum |
| --------------- | ------: |
| Service fit     |      30 |
| Problem clarity |      15 |
| **Total**       |  **45** |

The LLM also returns structured qualitative analysis including:

- Primary need
- Problem categories
- Recommended service
- Internal lead summary
- Client-facing summary
- Suggested solution areas
- Discovery focus

The model response is constrained using a strict JSON schema so downstream nodes receive predictable structured fields rather than free-form text.

---

## LLM Output Validation

Structured output reduces variability in response format, but the workflow does not assume that model-generated values are automatically valid.

Before AI-generated scores contribute to the final qualification score, the workflow validates that each value is numeric and within its permitted range.

```javascript
const serviceFitScore = Number(ai.service_fit_score);
const problemClarityScore = Number(ai.problem_clarity_score);

if (
  !Number.isFinite(serviceFitScore) ||
  serviceFitScore < 0 ||
  serviceFitScore > 30
) {
  throw new Error("Invalid service_fit_score returned by AI");
}

if (
  !Number.isFinite(problemClarityScore) ||
  problemClarityScore < 0 ||
  problemClarityScore > 15
) {
  throw new Error("Invalid problem_clarity_score returned by AI");
}
```

This prevents malformed or out-of-range model output from silently affecting the final score or downstream routing.

---

## Final Scoring & Routing

The deterministic and AI-generated scores are combined into a final qualification score.

That score determines the lead's qualification status and routing priority:

| Score    | Qualification Status | Routing Priority |
| -------- | -------------------- | ---------------- |
| 80–100   | Highly Qualified     | High             |
| 60–79    | Qualified            | Standard         |
| 40–59    | Needs Review         | Review           |
| Below 40 | Not Qualified        | Low              |

The workflow then uses conditional routing to send each lead through the appropriate downstream path.

### High Priority

High-priority opportunities proceed through CRM record creation, internal notification, and follow-up intended to move the opportunity toward discovery.

### Standard Priority

Qualified standard-priority opportunities follow a similar CRM and communication path while remaining distinguishable from the highest-priority opportunities.

### Review

Ambiguous opportunities are routed for human review rather than forcing an automated qualification decision.

### Low Priority

Leads that fall below the qualification threshold can still receive an acknowledgement without triggering the same internal sales workflow as qualified opportunities.

---

## CRM Integration

The workflow integrates with **HubSpot** to move qualification results into the CRM rather than leaving the analysis isolated inside the automation platform.

Depending on the routing path, the workflow can create or update records including:

- Contact
- Company
- Deal

Relevant CRM associations are maintained so qualification activity can become part of the broader opportunity lifecycle.

Environment-specific CRM identifiers have been removed from the public workflow export.

---

## Internal Notifications

Qualified and review-worthy opportunities can trigger **Slack notifications** containing relevant qualification context such as:

- Company
- Contact
- Qualification score
- Budget
- Timeline
- Primary need
- Recommended service
- Suggested next action

The goal is to surface enough context for a human reviewer to understand why the opportunity was routed without having to reconstruct the qualification process manually.

---

## Client-Facing Communication

External follow-up uses controlled email templates combined with selected structured output from the AI analysis.

The LLM does **not** generate the entire email.

Instead, it produces a constrained `client_facing_summary` that can be inserted into a predefined communication template.

The system prompt explicitly prevents this summary from:

- Revealing qualification scores
- Revealing qualification status
- Exposing internal routing or evaluation criteria
- Inventing facts not supplied by the prospect
- Making guarantees
- Finalizing a solution before discovery
- Including pricing or contract language
- Generating its own scheduling request or call to action

This allows AI-generated language to add useful context while deterministic templates retain control over the surrounding communication.

---

## Human-in-the-Loop Design

Human review is treated as part of the system architecture rather than as an exception to automation.

The workflow uses deterministic logic where rules are explicit, AI where qualitative interpretation adds value, and human intervention where ambiguity or business judgment remains important.

The LLM contributes analysis, but it does not independently control objective qualification criteria or routing thresholds.

This separation makes it easier to understand which decisions are being made by business rules, which involve probabilistic analysis, and where a person remains responsible for the next decision.

---

## Testing

The workflow was developed and tested using synthetic business lead data.

Testing included:

- Intake and field normalization
- Deterministic score calculation
- Structured LLM output
- AI score range validation
- Final qualification scoring
- Qualification routing
- Repeated end-to-end execution of the high-priority path
- HubSpot record creation and associations
- Slack notification delivery
- Client-facing summary quality and grounding
- Data flow across the primary workflow path

Additional qualification branches were configured as part of the workflow architecture.

Full tier-specific regression testing across every downstream branch would be a next step before production deployment.

---

## Security & Privacy

This repository contains a **sanitized public workflow export**.

Credentials, account identifiers, execution data, and environment-specific configuration have been removed or replaced before publication.

Sanitized elements include:

- Authentication credentials
- Credential identifiers
- CRM owner identifiers
- CRM stage identifiers
- Workspace-specific identifiers
- Slack channel identifiers
- Environment-specific webhook identifiers
- n8n instance and workflow metadata
- Pinned execution data

Synthetic data is used for demonstration and testing.

The workflow is provided primarily as an implementation artifact for reviewing architecture, business logic, AI configuration, routing, validation, and integration design.

Running the workflow requires configuring your own credentials and environment-specific values for the connected services.

---

## Tools & Technologies

### Workflow & Integration

- n8n
- Webhooks
- JSON
- APIs and platform integrations
- Conditional routing
- Data mapping

### AI

- OpenAI
- Structured outputs
- JSON Schema
- Prompt and system design
- LLM output validation
- AI-assisted qualitative analysis

### Business Systems

- HubSpot CRM
- Slack
- SMTP email

### Implementation

- Requirements translation
- Business-rule design
- Workflow architecture
- CRM integration
- Human-in-the-loop design
- Testing and validation
- User and operational workflow design

---

## Key Design Decisions

### Separate deterministic and probabilistic decisions

Budget, timeline, and project readiness are explicit business rules and do not require model interpretation.

Service fit and problem clarity involve qualitative reasoning, making them better candidates for structured LLM analysis.

Separating the two makes the workflow easier to inspect, test, and modify.

### Normalize before processing

The inbound form payload is transformed into a consistent internal structure before qualification begins.

This reduces downstream dependency on the original intake format and creates a clearer boundary between ingestion and business logic.

### Constrain AI output

The LLM returns a defined JSON structure rather than unrestricted prose.

Downstream nodes can therefore reference specific fields reliably.

### Validate before trusting

AI-generated numeric scores are explicitly validated before contributing to the final qualification result.

### Preserve human judgment

The Review path provides a deliberate destination for opportunities that should not be forced into an automated yes/no decision.

### Separate internal and external AI output

Internal analysis can contain implementation-oriented detail, while client-facing output is generated under a narrower set of communication constraints.

---

## Repository Contents

```text
.
├── README.md
└── ai-lead-qualification-workflow.json
```

`ai-lead-qualification-workflow.json` contains the sanitized n8n workflow export used for this case study.

---

## Project Status

**Portfolio prototype — complete**

The workflow demonstrates the implemented architecture, qualification logic, AI integration, validation strategy, routing design, CRM integration, and primary end-to-end execution path.

Before production deployment, additional work would include broader regression testing, environment configuration, monitoring, error-handling strategy, credential management, and operational hardening.
