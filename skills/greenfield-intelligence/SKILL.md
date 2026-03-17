---
name: greenfield-intelligence
description: >
  Load this skill automatically when the BRD describes a new product, MVP, startup,
  or greenfield project — detected by keywords like "new product", "MVP", "v1",
  "greenfield", "launch", "startup", "no existing system". Provides technology
  implication flagging, business assumption detection, scalability consideration
  patterns, and greenfield-specific gap detection. Never make final technology
  decisions — flag and ask.
---

# Greenfield Intelligence Skill

## Purpose
Augment standard BRD analysis with greenfield-specific intelligence. The agent uses this skill to detect architectural implications, flag business assumption risks, identify compliance pre-screening needs, and add `greenfield_notes` to requirements. This skill does NOT make technology decisions — it flags implications and generates structured questions for the architecture team.

---

## Technology Implication Mapping

When a non-functional requirement is extracted, scan it against this table and add the matching `greenfield_notes` string:

| Requirement signal | Greenfield note to add |
|---|---|
| Concurrent users > 1,000 | "Horizontal scaling required. Stateless architecture recommended. Consider: load balancer + auto-scaling group + stateless app servers." |
| Concurrent users > 10,000 | "High-concurrency architecture needed. Consider: microservices decomposition, database read replicas, CDN for static assets, caching layer (Redis/Memcached). Discuss with architecture team before DB schema design." |
| Real-time updates / live data | "Real-time delivery required. Technology category choice needed: WebSockets (bidirectional, stateful), Server-Sent Events (unidirectional, simpler), or polling (simplest, highest latency). P1 question: acceptable latency threshold?" |
| Offline capability / offline-first | "Offline-first architecture required. Implies: local storage strategy, sync conflict resolution, service worker for PWA or native app. Significantly increases complexity — flag for architecture decision." |
| Mobile app required | "Native vs cross-platform decision needed. Implications for API design (REST vs GraphQL for bandwidth efficiency), offline capability, push notifications, app store compliance. P1 question: iOS only, Android only, or both?" |
| Video / audio streaming | "Media streaming infrastructure required. CDN with edge caching strongly recommended. Adaptive bitrate streaming (HLS/DASH) for variable bandwidth. Storage costs scale with content volume." |
| AI / ML / personalization | "ML infrastructure required: model serving, training pipeline, data labeling, feature store. Cold start problem for new users. Consider: third-party ML APIs vs. custom model training tradeoff." |
| Multi-tenant SaaS | "Multi-tenancy architecture decision: shared schema (simpler, lower cost, harder isolation) vs. schema-per-tenant (better isolation, more complex) vs. instance-per-tenant (highest isolation, highest cost). P0 for data privacy requirements." |
| Data volume > 1TB | "Large data volume: relational DB may need sharding strategy. Consider: columnar storage for analytics, time-series DB if data is time-indexed, object storage for unstructured data." |
| Search functionality | "Search infrastructure decision: full-text search in relational DB (simpler) vs. dedicated search engine (Elasticsearch, OpenSearch — more powerful but operational overhead). P1: expected search query complexity?" |
| Email / notification system | "Transactional email requires dedicated service (SendGrid, SES, Postmark). In-app notifications require real-time channel. Push notifications require APNs/FCM. Each has distinct deliverability and compliance requirements." |
| Payment processing | "PCI-DSS compliance mandatory. Use a compliant payment processor (Stripe, Braintree, Square) rather than handling card data directly. Scope reduction through tokenization is standard practice." |
| File upload / storage | "File storage strategy: local disk (no — not scalable), object storage (S3/GCS/Azure Blob — recommended), CDN in front of object storage for read performance. Virus scanning required for user uploads." |
| API for third parties / public API | "API design: REST (universal), GraphQL (flexible client queries), gRPC (high performance, internal). Versioning strategy needed from day 1. Rate limiting and API key management required." |
| Compliance with data regulations | "Data residency decision needed: where is data stored? EU data subjects → GDPR → EU/EEA data residency or Standard Contractual Clauses required. Compliance-radar skill should be loaded." |

---

## Business Assumption Detection

Scan the BRD for stated metrics, projections, or assumptions. For each detected:

**Revenue/conversion projections** (e.g., "5% conversion rate", "$2M ARR"):
Generate P1 note: "BRD states `{metric}`. This business assumption drives feature scope and architecture cost. Please confirm this has been validated through market research or validated with comparable products before architecture commitments are made."

**User volume projections** (e.g., "10,000 users per district", "1M users by year 2"):
Generate P1 note: "BRD projects `{volume}` users. Architecture and infrastructure costs scale with this number. If actual growth differs by 10x either direction, architectural decisions made today may need to be revisited."

**Timeline assumptions** (e.g., "MVP by August", "go-live before school year"):
Generate P1 note: "Hard deadline `{date}` detected. This constrains which features can realistically be included in MVP. The following requirements may need to be deferred: [flag any high-complexity requirements]."

**Market share / competitive claims** (e.g., "capture 15% market share"):
Generate P2 note: "Competitive market claims in BRD are not validated by this analysis. Recommend validating against current market research before scope commitments."

---

## Scalability Consideration Patterns

For any requirement with performance or scale implications, add this to `greenfield_notes`:

**Pattern 1: Read-heavy workloads**
Signal: many users reading/viewing data, dashboards, reports
Note: "Read-heavy pattern detected. Consider: read replicas, query result caching, CDN for static data. Write path can be simpler."

**Pattern 2: Write-heavy workloads**
Signal: logging, event tracking, IOT data, user activity recording
Note: "Write-heavy pattern detected. Consider: append-only data structures, event sourcing, message queue (Kafka/SQS) to decouple write burst from processing."

**Pattern 3: Compute-intensive operations**
Signal: report generation, data export, video processing, ML inference
Note: "Compute-intensive operation detected. Should be async (background job) rather than synchronous request. Consider: job queue (Sidekiq, Celery, AWS SQS + Lambda)."

**Pattern 4: Session-sensitive operations**
Signal: shopping carts, multi-step forms, user workflows spanning multiple requests
Note: "Session state required. Stateless architecture needs external session store (Redis). If load balancer needed, sticky sessions or shared session storage required."

---

## Greenfield-Specific Gap Detection

In addition to standard gap analysis (Phase 7), check for these greenfield-specific gaps:

1. **No MVP scope defined**: BRD describes full product but does not identify which features are MVP vs. later phases. Generate P0 question: "Which requirements are MVP (required for initial launch) vs. Phase 2+? Please tag must-have requirements as MVP-critical."

2. **No technology constraints stated**: Greenfield projects need early technology decisions. Generate P1 question: "No technology stack constraints stated in BRD. For architecture phase: (a) are there team expertise constraints? (b) infrastructure preferences (cloud provider)? (c) budget constraints on licensing?"

3. **No definition of done**: Generate P1 question: "What constitutes a successful MVP launch? Please define: (a) minimum feature set, (b) performance benchmarks required at launch, (c) user acceptance criteria."

4. **Growth model undefined**: Generate P1 question: "The BRD mentions `{volume}` users but does not describe growth trajectory. Architecture must be designed for expected peak load within 18 months. What is the projected growth rate?"

5. **No data migration**: If brownfield signals detected alongside greenfield signals (contradictory classification), generate P0 question: "Is this a new product or a replacement for an existing system? If replacement: is there existing data to migrate?"

---

## Output: greenfield_guidance.json

Write this file to `/analysis/greenfield_guidance.json` after Phase 8:

```json
{
  "project_classification": "GREENFIELD",
  "technology_implications": [
    {
      "requirement_id": "REQ-NF-XXX",
      "signal_detected": "concurrent users > 10,000",
      "implication": "High-concurrency architecture needed...",
      "decision_needed": "Architecture team must decide: monolith vs microservices before schema design",
      "urgency": "before_architecture_phase"
    }
  ],
  "business_assumption_flags": [
    {
      "assumption": "5% conversion rate",
      "source_verbatim": "exact text from BRD",
      "validation_question": "Has this been validated against market research?",
      "risk_if_wrong": "Feature scope and infrastructure cost assumptions may be invalid"
    }
  ],
  "scalability_patterns": ["read-heavy", "write-heavy"],
  "greenfield_gaps": [
    {
      "gap_type": "no_mvp_scope",
      "question_id": "Q-P0-XXX",
      "blocking": true
    }
  ],
  "mvp_candidates": ["REQ-F-XXX"],
  "phase2_candidates": ["REQ-F-XXX"]
}
```
