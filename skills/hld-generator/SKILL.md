---
name: hld-generator
description: >
  Load this skill at Step 5 of the Architecture Design Agent (v3.2.2). Produces the
  High-Level Design document from domain_model.json + architecture_handoff.json.
  Every design decision must cite a requirement ID, NFR target, or explicit
  constraint. Architecture style (monolith/microservices/serverless) is determined
  only from explicit input — never assumed. Decision Log is mandatory.
---

# HLD Generator Skill

## Core Rule: Decision Citation Requirement

Every single design decision in the HLD must include a `justified_by` field citing one of:
- A requirement ID (`REQ-F-XXX` or `REQ-NF-XXX`)
- A business rule ID (`BR-XXX`)
- An explicit constraint from `architecture_handoff.json`
- A compliance requirement (regulation name + specific article/rule)

If a design choice cannot be justified by any of the above, it is an assumption and MUST NOT be included. The HLD Decision Log documents these justifications explicitly.

---

## Step 1: Architecture Style Selection

Determine the system architecture style ONLY from explicit inputs:

**Check these sources in order**:
1. `technology_constraints` in architecture_handoff.json — any constraint mentioning "monolith", "microservices", "serverless", "architecture style"
2. NFR targets — if concurrent users > 10,000 OR scalability_patterns includes "horizontal-scaling", the NFR itself implies stateless + horizontal scaling (not a style assumption — derive from the numeric NFR)
3. Integration count — if > 5 external integrations, each with independent failure modes, bounded contexts suggest service decomposition (derive from integration list, not assumption)
4. If none of the above specify a style — output as `"architecture_style": "UNDECIDED"` and add to ambiguity list

**Architecture style schema**:
```json
{
  "architecture_style": "modular-monolith | microservices | serverless | hybrid | UNDECIDED",
  "justified_by": ["REQ-NF-001 requires 5000 concurrent, NFR-011 requires 3x spike = 15000 — implies stateless horizontal scaling", "explicit constraint: Node.js (stateless by design)"],
  "alternative_considered": "microservices — rejected because team size and MVP timeline favor monolith-first",
  "migration_path": "Modular monolith at launch, extract high-load services (search, notifications) at 10k users"
}
```

---

## Step 2: Component Breakdown

Derive system components from bounded contexts in domain_model.json. One component per bounded context is the default. Split a context into multiple components only if justified by an NFR or explicit constraint.

**Component schema**:
```json
{
  "component_id": "COMP-001",
  "name": "VendorService",
  "type": "backend-service | frontend | background-worker | api-gateway | cache | queue | database | external",
  "bounded_context": "VendorContext",
  "responsibilities": ["Vendor registration", "Document verification", "Storefront management"],
  "exposes_apis": true,
  "consumes_apis": ["COMP-003"],
  "data_store": "COMP-010",
  "justified_by": ["REQ-F-001 through REQ-F-006 all relate to vendor management", "VendorContext in domain model"],
  "nfr_constraints": ["NFR-001: must support 5000 concurrent — stateless design required"]
}
```

**Standard components derived from ShopFlow-style requirements** (only include if corresponding bounded context exists):
- API Gateway — if multiple services AND external clients exist
- Auth Service — if NFR or FR explicitly requires authentication
- Search Service — if NFR-002 specifies sub-2-second search AND NFR-012 requires 10M listings scale (implies dedicated search engine, not DB)
- Notification Worker — if FR requires async email + SMS (background processing)
- Payment Service — if PCI-DSS compliance required (payment must be isolated)
- CDN — if NFR for Lighthouse score or media delivery exists

---

## Step 3: External Integration Mapping

For each integration in `context.integrations`, produce:

```json
{
  "integration_id": "INT-001",
  "external_system": "Razorpay",
  "integration_type": "payment-gateway",
  "consuming_component": "COMP-Payment",
  "communication_pattern": "synchronous-REST | async-webhook | polling | event-stream",
  "failure_handling": "timeout after 30s → mark payment pending (REQ-F-031)",
  "fallback": "route to PayU if Razorpay returns 5xx (explicit constraint Q-P2-001)",
  "auth_method": "API key — provided by Razorpay at onboarding",
  "data_flow": "payment_request → Razorpay → tokenized_response → store token only (FR-030, NFR-006)",
  "justified_by": ["REQ-F-029", "REQ-F-030", "REQ-F-031", "explicit constraint: PCI-DSS"]
}
```

---

## Step 4: NFR Mapping

For each NFR target in `context.nfr_targets`, map to the component(s) and design decisions that address it:

```json
{
  "nfr_req_id": "REQ-NF-001",
  "metric": "5000 concurrent users",
  "addressed_by_components": ["COMP-APIGateway", "COMP-AppServer"],
  "design_decisions": [
    "Stateless app servers — sessions in Redis (COMP-Redis)",
    "Horizontal auto-scaling via AWS ASG",
    "ALB for load distribution"
  ],
  "monitoring_metric": "active connections per server + ASG scaling events",
  "justified_by": "REQ-NF-001 explicit target + explicit constraint (AWS ap-south-1)"
}
```

---

## Step 5: Security Architecture

Derive security design from compliance requirements and NFRs only:

```json
{
  "auth_mechanism": "JWT + refresh tokens (stateless, required by horizontal scaling)",
  "justified_by": "NFR-001 stateless requirement + REQ-F-042 (Google OAuth mentioned) + REQ-F-043 (password rules)",
  "encryption_in_transit": "TLS 1.2+ required by NFR-006",
  "encryption_at_rest": "AES-256 for vendor financial data — NFR-010",
  "pci_scope_reduction": "Razorpay tokenization — FR-030. Platform never handles raw card data.",
  "rate_limiting": "20 login attempts per IP per 10 min — explicit constraint Q-P1-020",
  "account_lockout": "30 min after 5 failed attempts — explicit constraint Q-P1-020",
  "audit_logging": "all order/payout/account changes with before/after values — NFR-019 (7yr retention)"
}
```

---

## Step 6: Deployment Architecture

Only describe deployment if explicitly specified in technology_constraints. If AWS ap-south-1 is an explicit constraint:

```json
{
  "cloud_provider": "AWS",
  "region": "ap-south-1 (Mumbai)",
  "justified_by": "Explicit constraint Q-P1-001 + NFR-018 data residency",
  "services_used": [
    {"aws_service": "ALB", "for_component": "API Gateway", "justified_by": "NFR-001 load distribution"},
    {"aws_service": "EC2 Auto Scaling Group", "for_component": "App Servers", "justified_by": "NFR-011 3x spike handling"},
    {"aws_service": "RDS PostgreSQL", "for_component": "Primary database", "justified_by": "Explicit constraint Q-P1-001"},
    {"aws_service": "ElastiCache Redis", "for_component": "Session store + cache", "justified_by": "Explicit constraint Q-P1-001"},
    {"aws_service": "OpenSearch Service", "for_component": "Product search", "justified_by": "NFR-002 + NFR-012 — explicit constraint Q-P1-001"},
    {"aws_service": "S3", "for_component": "Product images + invoices", "justified_by": "invoice 7yr retention NFR-019"},
    {"aws_service": "SES", "for_component": "Transactional email", "justified_by": "FR-055 + explicit constraint Q-P1-001"},
    {"aws_service": "CloudFront CDN", "for_component": "Static assets + product images", "justified_by": "NFR-004 Lighthouse 75+ score"}
  ],
  "environment_separation": "dev / staging / prod — separate AWS accounts",
  "data_residency_confirmed": "All services in ap-south-1. CloudFront configured to cache at Indian edge locations only."
}
```

If cloud/deployment not specified — output `"deployment": "UNSPECIFIED — architecture agent requires explicit infrastructure decision before this section can be completed."` Do not invent a deployment.

---

## Step 7: Decision Log (MANDATORY)

Every decision in the HLD gets a log entry. Minimum one entry per: architecture style, each component creation, each integration mapping, each security decision, deployment choice.

```json
{
  "decision_id": "DEC-001",
  "decision": "Use modular monolith architecture at MVP",
  "justified_by": ["REQ-NF-001 5000 concurrent — achievable with monolith + horizontal scaling", "Explicit constraint: Node.js team (no microservices expertise)", "MVP timeline (Oct 2026) — microservices decomposition adds 3-4 months"],
  "alternatives_considered": [
    {"alternative": "Microservices from day 1", "rejected_because": "No team expertise per Q-P1-001. Timeline risk. Premature optimization at MVP scale."}
  ],
  "trade_offs": "Less operational complexity at MVP. Will require careful module boundary design to enable future extraction.",
  "reversibility": "High — modular boundaries allow service extraction without full rewrite",
  "reviewed_by": "Architecture agent — must be confirmed by human architect before implementation"
}
```

---

## HLD Output Files

**hld.json**: Machine-readable full HLD
**HLD.md**: Human-readable markdown version

Both must be produced. The markdown version is derived from hld.json — not written separately.

**HLD.md structure**:
```markdown
# High-Level Design — {Project Name}
**Version**: 1.0 | **Date**: {date} | **Status**: Draft

## Architecture overview
## System components
## Component diagram (text-based ASCII or description)
## External integrations
## Non-functional requirement mapping
## Security architecture
## Deployment architecture
## Decision log
## Future scope (Phase 2 components — not designed, acknowledged only)
```
