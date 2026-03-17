---
name: nfr-quantification
description: >
  Load this skill when non-functional requirements are detected. Forces numeric
  quantification on all vague NFRs by providing industry-standard defaults and
  asking targeted confirmation questions. Prevents downstream architecture agents
  from receiving qualitative-only NFRs like "fast" or "scalable" that cannot be
  used for infrastructure sizing, SLA definition, or performance test design.
---

# NFR Quantification Skill

## Purpose
Every non-functional requirement (NFR) must have a numeric target before it reaches the architecture agent. Vague NFRs like "the system shall be fast" or "must be highly available" are unusable for: infrastructure sizing, SLA definition, capacity planning, performance test creation, and monitoring alert configuration. This skill forces quantification by applying industry-standard defaults and asking the user to confirm or adjust.

---

## Step 1: Identify Unquantified NFRs

Read from `/analysis/requirements_catalog.json`. Filter for requirements where:
- `type` = `"non-functional"` AND
- `nfr_quantified_target` = `null` AND
- No numeric value appears in `description` or `acceptance_criteria`

Also flag functional requirements that have **implicit NFR aspects**:
- "System shall search products" → implies search response time target
- "System shall support 5000 users" → implies concurrent user capacity target
- "Dashboard shall load" → implies page load time target

---

## Step 2: Apply Industry-Standard Defaults

For each unquantified NFR, look up the appropriate default based on category and domain:

### Performance Defaults
| Metric | Domain | Default | Source |
|---|---|---|---|
| API response time | E-commerce | < 300ms p95 | Google RAIL model |
| API response time | Enterprise SaaS | < 500ms p95 | Industry standard |
| Page load time | Consumer web | < 3s (First Contentful Paint) | Google Core Web Vitals |
| Page load time | Enterprise app | < 5s (interactive) | Industry standard |
| Search response | E-commerce | < 200ms p95 | Amazon/Google benchmarks |
| Search response | Enterprise | < 500ms p95 | Industry standard |
| File upload | Any | < 10s for 10MB file | UX best practice |
| Real-time update | Chat/collaboration | < 100ms | WhatsApp/Slack benchmark |
| Real-time update | Dashboard/monitoring | < 2s | Industry standard |
| Batch processing | Any | Define completion SLA (e.g., < 1 hour for 1M records) | Case-specific |

### Availability Defaults
| Metric | Tier | Default | Meaning |
|---|---|---|---|
| Uptime | Critical (payments, auth) | 99.99% (52m downtime/year) | Four nines |
| Uptime | Standard (main app) | 99.9% (8.7h downtime/year) | Three nines |
| Uptime | Non-critical (reports, admin) | 99.5% (1.8d downtime/year) | Standard |
| Recovery time (RTO) | Critical | < 1 hour | Rapid recovery |
| Recovery time (RTO) | Standard | < 4 hours | Standard recovery |
| Recovery point (RPO) | Critical | < 1 minute | Near-zero data loss |
| Recovery point (RPO) | Standard | < 1 hour | Acceptable loss |

### Scalability Defaults
| Metric | Stage | Default |
|---|---|---|
| Concurrent users | MVP | 500 |
| Concurrent users | Growth | 5,000 |
| Concurrent users | Scale | 50,000+ |
| Data storage growth | Year 1 | Estimate based on entity count × expected records |
| API requests/second | MVP | 100 rps |
| API requests/second | Scale | 10,000 rps |

### Security Defaults
| Metric | Default |
|---|---|
| Password hashing | bcrypt with cost factor ≥ 12 |
| Session timeout | 30 minutes idle, 24 hours absolute |
| Token expiry | Access: 15 min, Refresh: 7 days |
| Rate limiting | 100 requests/minute per user, 1000/minute per IP |
| Data encryption at rest | AES-256 |
| Data encryption in transit | TLS 1.2+ |

---

## Step 3: Generate Quantification Questions

For each unquantified NFR, generate a targeted question:

**Question format**:
```
[REQ-NF-XXX] "{requirement_title}" has no numeric target.

Current text: "{source_verbatim}"

Suggested target based on [{domain}] industry standard:
  → {metric}: {default_value} {unit} ({justification})

Options:
  [1] Accept suggested default: {default_value} {unit}
  [2] Set custom value: _____
  [3] This is not a concern for MVP — defer to architecture team

Which do you prefer?
```

**Group questions by category** to minimize user interaction:
- All performance questions together
- All availability questions together
- All scalability questions together

---

## Step 4: Record Quantified Targets

For each NFR, populate the `nfr_quantified_target` field:

```json
{
  "metric": "response_time | uptime | concurrent_users | throughput | recovery_time | storage_growth | page_load_time",
  "value": 300,
  "unit": "ms | percent | users | rps | hours | GB_per_month | seconds",
  "percentile": "p50 | p95 | p99 | null",
  "source": "brd_explicit | user_confirmed | industry_default_confirmed | industry_default_unconfirmed | deferred_to_architecture",
  "default_used": "< 300ms p95 (e-commerce API standard)",
  "confirmed_by_user": true
}
```

**Source values**:
- `brd_explicit`: BRD already contained a numeric target — just extracted it
- `user_confirmed`: User accepted or modified the suggested default
- `industry_default_confirmed`: User explicitly accepted the industry default
- `industry_default_unconfirmed`: User did not respond — default applied but flagged for review
- `deferred_to_architecture`: User chose to let architecture team decide

---

## Step 5: Update Requirements

Write the `nfr_quantified_target` back to each requirement in `/analysis/requirements_catalog.json` using `vscode/str_replace`.

If the user did NOT confirm (source = `industry_default_unconfirmed`):
- Set `clarification_needed: true`
- Add to `/analysis/pending_clarifications.json`
- Note: "NFR target set to industry default but not confirmed by user. Architecture team should validate before committing to SLA."

---

## Validation Rules

- Every NFR must have `nfr_quantified_target` populated after this skill runs (no nulls remaining)
- `value` must be a number, not a string
- `unit` must be from the allowed set
- `source` must accurately reflect how the target was determined
- If `source` = `industry_default_unconfirmed`, then `clarification_needed` must be `true`
