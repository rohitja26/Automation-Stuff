---
name: compliance-radar
description: >
  Load this skill automatically when BRD signals regulated data: personal data,
  health/medical information, financial data, children's data, educational records,
  government systems, or public-facing accessibility requirements. Maps requirements
  to applicable regulations, detects compliance gaps, and generates compliance
  requirement suggestions. Never invents compliance requirements — only flags where
  BRD implies regulated data handling.
---

# Compliance Radar Skill

## Purpose
Perform systematic compliance pre-screening during BRD analysis. The agent uses this skill to map detected regulated data types to applicable regulations, identify which requirements have compliance coverage, find gaps where regulated data is implied but no compliance requirement exists, and generate precise compliance questions and gap-fill suggestions.

---

## Regulation Detection Signals

Scan all extracted content and requirement descriptions for these signals:

| Signal keywords | Regulations triggered |
|---|---|
| children, minors, under 13, K-12, students under 13, school | COPPA, FERPA (if educational), SOPIPA (CA) |
| student records, grades, transcripts, educational institution, teacher, school district | FERPA |
| health, medical, patient, diagnosis, prescription, clinical, healthcare, PHI, EHR, EMR | HIPAA, HITECH |
| EU, European, GDPR, data subject, right to erasure, right to access, consent | GDPR |
| personal data, PII, name + address, email + dob, location data | GDPR (if EU), CCPA (if CA), applicable state laws |
| California users, CA residents | CCPA / CPRA |
| payment, credit card, card number, CVV, billing, PCI | PCI-DSS |
| financial records, bank account, investment, loan, credit report | GLBA, SOX (if publicly traded) |
| federal, government, federal agency, .gov | FedRAMP, FISMA, NIST SP 800-53 |
| accessibility, disability, screen reader, WCAG, ADA, Section 508 | WCAG 2.1 AA, ADA Title III, Section 508 |
| HR, employee data, employment records, payroll | GDPR (if EU employees), applicable labor laws |
| biometric, fingerprint, face recognition, retina, voiceprint | BIPA (IL), GDPR Article 9 |

---

## Regulation Profiles

### GDPR (General Data Protection Regulation)
**Scope**: Any processing of personal data of EU/EEA data subjects, regardless of where the processor is located.
**Key requirements to check**:
- [ ] Lawful basis for processing identified (consent, legitimate interest, contract, legal obligation)
- [ ] Data subject rights implemented: access, erasure, portability, rectification, restriction, objection
- [ ] Data minimization principle: only collect what is necessary
- [ ] Purpose limitation: data used only for stated purposes
- [ ] Storage limitation: retention periods defined
- [ ] Privacy by design: privacy considered in architecture
- [ ] Data breach notification (72-hour requirement to supervisory authority)
- [ ] Data Protection Impact Assessment (DPIA) if high-risk processing
- [ ] Data Processing Agreements with processors

**Gap question template**: "Requirement `{REQ-ID}` processes `{data type}` for EU users but no GDPR compliance requirement exists. Should requirements be added for: (a) lawful basis documentation, (b) data subject rights implementation, (c) data retention limits?"

---

### COPPA (Children's Online Privacy Protection Act)
**Scope**: Online services directed at children under 13, or with actual knowledge of collecting data from children under 13 (US law, applies to services accessible in the US).
**Key requirements to check**:
- [ ] Age verification or age-gating mechanism
- [ ] Verifiable parental consent before collecting personal info from under-13 users
- [ ] Parental review and deletion rights
- [ ] No behavioral advertising to children
- [ ] Data minimization for children's data
- [ ] Privacy notice written for parents (not children)

**Gap question template**: "BRD targets users that may include children under 13. No COPPA compliance requirements detected. Must-have requirements to add: (a) age verification mechanism, (b) verifiable parental consent flow, (c) data minimization for minors."

---

### FERPA (Family Educational Rights and Privacy Act)
**Scope**: Educational institutions receiving federal funding. Protects student education records.
**Key requirements to check**:
- [ ] Student record access controls (students and parents only, or authorized school officials)
- [ ] No disclosure to third parties without consent (or FERPA exception)
- [ ] Annual notification to students/parents of FERPA rights
- [ ] Directory information designation and opt-out
- [ ] Legitimate educational interest controls for school officials

**Gap question template**: "BRD handles student educational records (FERPA-covered). No access control requirements for record disclosure detected. Should requirements be added for: (a) role-based access to student records, (b) consent tracking for third-party disclosures?"

---

### HIPAA (Health Insurance Portability and Accountability Act)
**Scope**: Covered entities (healthcare providers, insurers, clearinghouses) and their business associates. Protects Protected Health Information (PHI).
**Key requirements to check**:
- [ ] PHI encryption at rest and in transit
- [ ] Access controls and audit logs for PHI access
- [ ] Minimum necessary standard (access only to PHI needed for the task)
- [ ] Business Associate Agreements (BAA) with vendors who access PHI
- [ ] Breach notification (60-day requirement to HHS)
- [ ] Patient right of access to their PHI
- [ ] Disposal of PHI (secure deletion)

**Gap question template**: "Requirement `{REQ-ID}` handles health data that may constitute PHI under HIPAA. No encryption or access control requirement found. Should requirements be added for: (a) PHI encryption at rest (AES-256), (b) audit logging of PHI access, (c) BAA requirement for vendors?"

---

### PCI-DSS (Payment Card Industry Data Security Standard)
**Scope**: Any entity that stores, processes, or transmits cardholder data.
**Key requirements to check**:
- [ ] No storage of sensitive authentication data post-authorization (CVV, full track data, PIN)
- [ ] Cardholder data encrypted in transit (TLS 1.2+)
- [ ] Scope reduction via tokenization (preferred over storing card data)
- [ ] Use of PCI-compliant payment processor
- [ ] Quarterly vulnerability scans
- [ ] Annual penetration testing

**Gap question template**: "BRD includes payment functionality. No PCI-DSS compliance requirements detected. Strongly recommended: (a) use a PCI-compliant payment processor (Stripe, Braintree) rather than handling card data directly, (b) confirm scope reduction via tokenization, (c) add requirement for TLS 1.2+ on all payment flows."

---

### WCAG 2.1 AA (Web Content Accessibility Guidelines)
**Scope**: Public-facing web applications. Required by ADA Title III for public accommodations, Section 508 for US federal systems.
**Key requirements to check**:
- [ ] All images have alt text
- [ ] Color is not the only means of conveying information
- [ ] Minimum contrast ratio (4.5:1 for normal text, 3:1 for large text)
- [ ] All functionality accessible by keyboard
- [ ] Focus indicators visible
- [ ] Form labels associated with inputs
- [ ] Error messages descriptive and accessible
- [ ] Videos have captions
- [ ] Content readable at 200% zoom

**Gap question template**: "BRD describes a public-facing interface but no accessibility requirements are stated. WCAG 2.1 AA is required for ADA compliance. Should requirements be added for: (a) keyboard navigation, (b) screen reader compatibility, (c) color contrast ratios?"

---

## Compliance Gap Analysis Process

**Step 1**: For each regulation triggered by signals, map it to the requirements that handle the relevant data type.

**Step 2**: For each mapped regulation, check if a requirement exists that addresses each key item in the regulation's checklist above.

**Step 3**: For each key item with no coverage, generate a compliance gap entry:
```json
{
  "regulation": "GDPR",
  "requirement_area": "data subject rights",
  "gap": "No requirement for right-to-erasure (right to be forgotten)",
  "related_requirements": ["REQ-F-015", "REQ-NF-003"],
  "suggested_requirement": "REQ-F-XXX: The system shall allow users to request deletion of all their personal data, with deletion completed within 30 days and confirmation sent to the user.",
  "urgency": "must_address_before_launch",
  "question_id": "Q-P0-XXX"
}
```

**Step 4**: Flag each gap as:
- `must_address_before_launch` — legal requirement with no flexibility
- `strongly_recommended` — best practice, legal risk if omitted
- `recommended` — good practice, lower immediate risk

---

## Adding Compliance Flags to Requirements

For every requirement that handles regulated data:
1. Add `compliance_flags` array to the requirement (e.g., `["GDPR", "COPPA"]`)
2. Add a note in `greenfield_notes` (if greenfield) or in requirement description: "This requirement handles `{data type}` subject to `{regulation}`. Ensure compliance requirement REQ-C-XXX is addressed before implementation."

**Compliance requirement IDs**: Use prefix `REQ-C-XXX` for requirements that exist solely for compliance (not functional features). These are added to requirements_catalog.json as compliance-type requirements.

---

## Output: compliance_radar section in assumptions_and_risks.json

Append this section to `/analysis/assumptions_and_risks.json`:

```json
{
  "compliance_radar": {
    "regulations_applicable": ["GDPR", "COPPA", "FERPA"],
    "detection_basis": {
      "GDPR": "BRD mentions EU users and personal data collection",
      "COPPA": "Platform serves K-12 students (potentially under 13)",
      "FERPA": "Student educational records managed by platform"
    },
    "coverage_by_regulation": {
      "GDPR": {
        "covered_items": ["data encryption", "user consent"],
        "gap_items": ["right to erasure", "data portability", "DPIA", "data retention limits"],
        "coverage_percentage": 25
      }
    },
    "compliance_gaps": [],
    "suggested_compliance_requirements": [],
    "overall_compliance_readiness": "low | medium | high"
  }
}
```
