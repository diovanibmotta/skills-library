# aws-well-architected-review

Acts as a senior Cloud Engineer/Solutions Architect to evaluate an AWS solution architecture (IaC code, a diagram, a design document, or a plain description) against the AWS Well-Architected Framework, with primary focus on Reliability (resilience, availability, fault tolerance) and Security.

## Trigger

```
/aws-well-architected-review
```

## What it does

1. Determines the review target (existing IaC module, an architecture still being designed, or a service already in production) and its scope
2. Gathers non-functional requirements (criticality, RTO/RPO, sensitive data) or explicitly states the production-default assumption when they're missing
3. Builds a component/AWS-service inventory by actually inspecting the code/diagram/description
4. Applies the Reliability and Security pillar checklists in depth (foundations, workload architecture, change management, failure management; identity, detection/traceability, network protection, data protection)
5. Walks through single points of failure and blast radius component by component
6. Applies a lighter pass over Operational Excellence, Performance Efficiency, Cost Optimization, and Sustainability, unless the user asks for a full six-pillar review
7. Classifies every finding on a 🔴 CRITICAL → ⚪ INFO risk scale, tagged by pillar, with a concrete AWS service/pattern recommendation
8. Generates a versioned report at `.ai/architecture-reviews/<slug>/review-<timestamp>.md`
9. Presents a chat summary: findings table, top 3 priorities, assumptions made, and an escalation recommendation when warranted

## Target stack

AWS specifically (service names, patterns, and the disaster-recovery strategy table are AWS-specific) — any workload type (Terraform/CloudFormation IaC, a design still on paper, or an already-running service).

## Usage

```
/aws-well-architected-review
/aws-well-architected-review infra/payments/aws
```

## Notes

- Complements (does not replace) the `terraform-iac` skill (code writing conventions) and the `security-verify` skill (point-in-time vulnerabilities in a diff) — this skill's focus is system-level architecture decisions.
- Deliberate, documented trade-offs (e.g. a dev environment without Multi-AZ to save cost) are treated as accepted decisions, not findings.
- Not a substitute for a formal AWS Well-Architected Review conducted with an AWS or partner Solutions Architect.
