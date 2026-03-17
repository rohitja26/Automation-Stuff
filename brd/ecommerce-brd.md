# Business Requirements Document
## ShopFlow — Multi-Vendor E-Commerce Platform

**Document version**: 1.1  
**Status**: Draft — operational requirements added post-analysis  
**Date**: March 2026  
**Last updated**: March 16, 2026 — Added Section 8.7 Operational Requirements (NFR-021 to NFR-029)  
**Project classification**: Greenfield / New product (MVP launch target: Q4 2026)  
**Prepared by**: Product Team  

---

## 1. Executive Summary

ShopFlow is a new multi-vendor e-commerce platform targeting small-to-medium independent retailers in the Indian market. The platform enables vendors to create and manage their own storefronts under a unified marketplace, while customers get a single destination for discovering and purchasing products across multiple sellers.

This document defines the business requirements for the MVP. It does not describe technical implementation — that is the responsibility of the architecture team following BRD Maturity Scoring (Stage 2).

---

## 2. Business Objectives

The platform must achieve the following measurable outcomes within 18 months of launch:

1. Onboard 500 active vendors by end of month 6
2. Achieve 10,000 monthly active customers by end of month 12
3. Process ₹5 crore in gross merchandise value (GMV) per month by month 18
4. Maintain a vendor satisfaction score of 4.2/5.0 or higher (measured via quarterly survey)
5. Achieve a customer repeat purchase rate of 35% or higher by month 12

**Business model**: Commission-based. ShopFlow charges vendors 8% commission on each completed transaction. Vendors also pay a monthly subscription fee of ₹999/month for the Professional tier or ₹499/month for the Basic tier.

---

## 3. Stakeholders

| Name | Role | Responsibilities | Decision authority |
|---|---|---|---|
| Priya Sharma | Chief Product Officer | Product vision, feature prioritization, go-to-market | High |
| Rohit Mehta | CTO | Technology architecture, scalability planning, team | High |
| Ananya Kapoor | Head of Vendor Relations | Vendor onboarding, vendor experience, support | Medium |
| Vikram Singh | Head of Customer Experience | Customer journey, UX standards, support escalation | Medium |
| Legal & Compliance Team | Compliance | GST compliance, payment regulations, data privacy | High (compliance decisions) |
| Target Vendors | Primary sellers | Inventory management, order fulfillment, pricing | Low (feedback only) |
| Target Customers | End buyers | Product discovery, purchase, returns | Low (feedback only) |

---

## 4. Scope

### 4.1 In Scope (MVP)

- Vendor registration, onboarding, and storefront creation
- Product catalog management (create, edit, publish, unpublish)
- Customer-facing product discovery and search
- Shopping cart and checkout flow
- Payment processing (UPI, credit/debit cards, net banking, cash on delivery)
- Order management for vendors and customers
- Basic inventory tracking with low-stock alerts
- Customer account management (registration, login, address book, order history)
- Vendor dashboard with sales analytics
- Commission calculation and vendor payout processing
- Customer reviews and ratings
- Basic promotional tools (discount codes, product-level offers)
- Mobile-responsive web interface
- Email and SMS notification system
- GST invoice generation

### 4.2 Out of Scope (MVP — planned for Phase 2)

- Native mobile applications (iOS and Android)
- Multi-language support beyond English and Hindi
- Advanced analytics and business intelligence
- Vendor advertising and sponsored listings
- Subscription box / recurring order functionality
- International shipping and multi-currency support
- Loyalty points and reward programs
- Live chat customer support
- Vendor-to-vendor messaging

---

## 5. Assumptions

- All vendors are GST-registered businesses (or individual sellers with valid PAN)
- Payment gateway partner (Razorpay or PayU) will be contracted before development begins
- Logistics integration will be limited to Shiprocket API for MVP
- The platform will serve Indian customers only in MVP — no international addresses
- Vendors are responsible for their own product photography and descriptions
- Customer support will be handled via email and phone (no in-app chat in MVP)

---

## 6. Constraints

- **Budget**: MVP development budget is ₹1.2 crore
- **Timeline**: MVP must be live before Diwali 2026 shopping season (October 2026)
- **Regulatory**: Platform must comply with IT Act 2000, Consumer Protection (E-Commerce) Rules 2020, and applicable GST regulations
- **Data residency**: All customer and transaction data must be stored on servers located within India (data localization requirement)
- **Payment compliance**: All payment flows must comply with RBI guidelines and PCI-DSS standards
- **Accessibility**: Platform must meet WCAG 2.1 AA accessibility standards for web interface

---

## 7. Functional Requirements

### 7.1 Vendor Registration and Onboarding

**FR-001**: The system shall allow new vendors to register by providing business name, GST number, PAN, bank account details, and contact information.

**FR-002**: The system shall validate GST numbers against the GSTN API before completing vendor registration.

**FR-003**: The system shall require vendors to upload the following documents during onboarding: GST registration certificate, cancelled cheque or bank statement, and one government-issued identity proof.

**FR-004**: The system shall notify the vendor operations team when a new vendor application is submitted, requiring manual review within 2 business days.

**FR-005**: The system shall send the vendor an automated approval or rejection email within 24 hours of the operations team completing review.

**FR-006**: Upon approval, the system shall automatically create a vendor storefront with a unique URL in the format `shopflow.in/{vendor-slug}`.

**Acceptance criteria for FR-001 through FR-006**:
- Given a new vendor completes the registration form, when they submit, then the system validates all mandatory fields before proceeding
- Given a vendor submits a GST number, when validation is called, then the system calls GSTN API and rejects the registration if the GST number is inactive or invalid
- Given an operations team member approves a vendor, when the approval is saved, then the vendor receives an email within 15 minutes and their storefront URL becomes accessible

---

### 7.2 Product Catalog Management

**FR-007**: The system shall allow vendors to create product listings with the following mandatory fields: product name, description (minimum 50 characters), category (from a fixed taxonomy), price, stock quantity, and at least one product image.

**FR-008**: The system shall allow vendors to create product variants (e.g., size, color) with individual pricing and stock quantities per variant.

**FR-009**: The system shall allow vendors to set products as active, inactive, or out-of-stock, with visibility changes taking effect within 60 seconds on the customer-facing site.

**FR-010**: The system shall allow vendors to bulk-upload products via CSV template. The CSV template must be downloadable from the vendor dashboard.

**FR-011**: The system must support a minimum of 10,000 active product listings per vendor without performance degradation.

**FR-012**: The system shall generate and display a unique product page URL for each active product listing.

**FR-013**: As a vendor, I want to be notified automatically when any product variant's stock quantity falls below a threshold I define, so that I can reorder before running out of stock.

**Acceptance criteria for FR-013**:
- Given a vendor sets a low-stock threshold of N units, when any variant reaches or falls below N units, then the vendor receives an email and dashboard notification within 30 minutes
- Given a vendor has not set a threshold, when stock falls below 5 units (default threshold), then the vendor receives the notification

---

### 7.3 Customer-Facing Product Discovery

**FR-014**: The system must provide a search function that returns relevant results within 2 seconds for queries up to 100 characters.

**FR-015**: The system shall support filtering search results by category, price range, vendor, rating, and availability.

**FR-016**: The system shall display product listings with the following information: product name, primary image, price, vendor name, average rating, and in-stock status.

**FR-017**: The system shall support product sorting by relevance, price (low to high, high to low), newest, and customer rating.

**FR-018**: The system must display search results pages with a maximum of 40 products per page, with pagination controls.

**FR-019**: As a customer, I want to save products to a wishlist, so that I can return to them later without searching again.

**FR-020**: The system shall display "similar products" recommendations on each product detail page. The recommendation logic will be defined by the engineering team — the business requirement is that at least 4 relevant products are shown.

---

### 7.4 Shopping Cart and Checkout

**FR-021**: The system must allow customers to add products from multiple vendors to a single cart.

**FR-022**: The system shall persist cart contents for logged-in customers across sessions, for a minimum of 30 days.

**FR-023**: The system shall display a real-time price breakdown in the cart including: item prices, shipping estimates, applicable GST, discount codes applied, and order total.

**FR-024**: The system must process checkout in a maximum of 3 steps: (1) address confirmation, (2) payment method selection, (3) order review and confirmation.

**FR-025**: The system shall support the following payment methods at launch: UPI (via payment gateway), credit card, debit card, net banking, and cash on delivery (COD). COD must be available as an option.

**FR-026**: The system shall validate delivery addresses against supported pin codes before allowing checkout completion.

**FR-027**: The system must display an order confirmation page and send an order confirmation email within 2 minutes of successful payment.

**FR-028**: The system shall split multi-vendor orders into separate sub-orders per vendor, each with its own tracking and fulfillment workflow, while presenting the customer with a single unified order view.

**Acceptance criteria for FR-021 through FR-028**:
- Given a customer adds products from 3 different vendors, when they view their cart, then all items are shown together with per-vendor subtotals visible
- Given a customer applies a discount code, when the code is valid, then the discount is reflected immediately in the price breakdown without requiring page reload
- Given a customer completes payment, when payment is confirmed by the payment gateway, then the order confirmation email is sent within 2 minutes and the order appears in the customer's order history

---

### 7.5 Payment Processing

**FR-029**: The system must integrate with a PCI-DSS compliant payment gateway (Razorpay preferred) to process all card and UPI payments.

**FR-030**: The system shall never store raw card numbers, CVV, or sensitive authentication data. All payment tokenization must be handled by the payment gateway.

**FR-031**: The system must handle payment gateway timeout scenarios gracefully: if no response within 30 seconds, the order must be flagged as "payment pending" and the customer notified to check their payment status before placing a new order.

**FR-032**: The system shall generate a GST-compliant invoice for every completed order, available for download by both the customer and the vendor.

**FR-033**: The system must calculate and apply the correct GST rate based on the product's HSN code (provided by the vendor during product creation).

**FR-034**: The system shall process vendor payouts on a weekly cycle (every Tuesday), transferring the vendor's earned amount minus ShopFlow's commission to the vendor's registered bank account via NEFT/IMPS.

**FR-035**: The system must hold payout for any order until the return window has closed (7 days after delivery confirmation) or the return is resolved.

---

### 7.6 Order Management

**FR-036**: The system shall provide vendors with a dashboard showing all orders in statuses: New, Confirmed, Packed, Shipped, Delivered, Cancelled, Return Requested, Return Completed.

**FR-037**: The system shall allow vendors to update order status and add tracking information (courier name and AWB number).

**FR-038**: The system must send automated notifications to customers at the following order events: order confirmed, order shipped (with tracking link), order delivered, return approved, refund processed.

**FR-039**: As a vendor, I want to be able to cancel an order before it is shipped, so that I can handle situations where the item is found to be damaged or out of stock after order receipt.

**FR-040**: The system shall allow customers to initiate a return request within 7 days of delivery for any order. The return reason must be selected from a predefined list.

**FR-041**: The system must process refunds to the original payment method within 5–7 business days of return approval. For COD orders, refunds must be processed via bank transfer.

---

### 7.7 Customer Account Management

**FR-042**: The system shall allow customers to register using email address and password, or via Google OAuth or phone number (OTP-based).

**FR-043**: The system must enforce password requirements: minimum 8 characters, at least one uppercase letter, one number, and one special character.

**FR-044**: The system shall allow customers to save multiple delivery addresses and designate one as default.

**FR-045**: The system shall display complete order history with status, items, amounts, and invoice download for each order.

**FR-046**: The system must allow customers to permanently delete their account. On deletion, all personal data must be removed from active systems within 30 days. Transaction records required for GST compliance may be retained in anonymized form.

---

### 7.8 Reviews and Ratings

**FR-047**: The system shall allow customers to leave a rating (1–5 stars) and written review for any product they have purchased and received.

**FR-048**: The system must verify that the reviewer has a confirmed purchase of the reviewed product before allowing a review to be submitted. Unverified reviews are not permitted.

**FR-049**: The system shall display average ratings and total review count on product listings and product detail pages.

**FR-050**: The system shall allow vendors to respond to customer reviews publicly. Vendor responses must not exceed 500 characters.

**FR-051**: The system must flag reviews containing prohibited content (defined by a moderation policy) for manual review before publication. Reviews not flagged must be published within 2 hours of submission.

---

### 7.9 Promotions and Discounts

**FR-052**: The system shall allow vendors to create discount codes with the following configuration options: percentage discount or fixed amount discount, minimum order value, usage limit (total uses and per-customer limit), and validity period.

**FR-053**: The system shall allow vendors to set sale prices on individual products or variants, with the original price displayed alongside the sale price.

**FR-054**: ShopFlow administrators must be able to create platform-wide discount campaigns that apply across all vendors (e.g., seasonal sales). Revenue impact of platform campaigns must be shared between ShopFlow and participating vendors per an agreed split — the exact split is TBD and must be defined before development.

---

### 7.10 Notifications

**FR-055**: The system must send transactional emails for the following events: vendor registration approval/rejection, new order received (vendor), order status updates (customer), low stock alerts (vendor), payout processed (vendor), return request received (vendor), refund confirmed (customer).

**FR-056**: The system must send SMS notifications for: order confirmation (customer), order shipped with tracking (customer), OTP for phone-based registration and login.

**FR-057**: The system shall allow customers to manage their notification preferences — they must be able to opt out of marketing emails while retaining transactional notifications. Transactional notifications cannot be disabled.

---

## 8. Non-Functional Requirements

### 8.1 Performance

**NFR-001**: The platform must support 5,000 concurrent users without performance degradation on page load times.

**NFR-002**: Product search results must be returned within 2 seconds for 95% of queries under normal load.

**NFR-003**: The checkout flow must complete (from cart to order confirmation) within 10 seconds for 95% of transactions, excluding payment gateway processing time.

**NFR-004**: Product pages must achieve a Google Lighthouse performance score of 75 or above on mobile.

**NFR-005**: The platform must maintain 99.9% uptime during business hours (6 AM – 12 AM IST). Planned maintenance windows must occur between 2 AM – 5 AM IST with 48 hours notice to vendors.

### 8.2 Security

**NFR-006**: All data transmission between client and server must use TLS 1.2 or higher.

**NFR-007**: The platform must implement rate limiting on all authentication endpoints to prevent brute-force attacks. Accounts must be locked after 5 consecutive failed login attempts.

**NFR-008**: All vendor and customer passwords must be stored using bcrypt or Argon2 hashing with appropriate work factors. Plain-text password storage is strictly prohibited.

**NFR-009**: The platform must pass an OWASP Top 10 security audit before launch. Critical and high vulnerabilities must be resolved before go-live.

**NFR-010**: Vendor financial data (bank account numbers, payout history) must be encrypted at rest using AES-256.

### 8.3 Scalability

**NFR-011**: The architecture must support horizontal scaling to handle 3x traffic spikes during peak events (Diwali, Big Billion Day equivalent campaigns).

**NFR-012**: The database design must support a catalog of 10 million product listings without requiring architectural changes.

### 8.4 Compliance

**NFR-013**: The platform must comply with the Information Technology (Reasonable Security Practices and Procedures and Sensitive Personal Data or Information) Rules, 2011 (SPDI Rules).

**NFR-014**: The platform must comply with Consumer Protection (E-Commerce) Rules, 2020, including mandatory display of country of origin, seller information, and grievance officer contact on all product pages.

**NFR-015**: The platform must implement a grievance redressal mechanism with a designated Grievance Officer. Customer complaints must receive an acknowledgement within 48 hours and resolution within 1 month per IT Act requirements.

**NFR-016**: All GST calculations must be accurate to the nearest rupee and compliant with current GST rate schedules. The platform must support HSN code-based GST rate lookup.

### 8.5 Accessibility

**NFR-017**: The customer-facing web interface must comply with WCAG 2.1 Level AA standards. This includes but is not limited to: all images having alt text, all forms being keyboard navigable, minimum contrast ratio of 4.5:1 for normal text, and all interactive elements having visible focus indicators.

### 8.6 Data and Privacy

**NFR-018**: Personal data of Indian users must be stored exclusively on servers located within India.

**NFR-019**: The platform must maintain an audit log of all changes to order status, vendor payouts, and customer account data. Audit logs must be retained for 7 years for GST compliance purposes.

**NFR-020**: The platform must implement a data retention policy: inactive customer accounts (no activity for 3 years) must be flagged for deletion review. Transaction data required for tax compliance must be retained per applicable law (currently 7 years).

### 8.7 Operational Requirements

**NFR-021**: The system shall return standardized HTTP error responses with error codes for all API failures:
- 400 Bad Request: Validation errors with field-level details
- 401 Unauthorized: Invalid or expired authentication token
- 403 Forbidden: Insufficient permissions for requested action
- 404 Not Found: Requested resource does not exist
- 409 Conflict: Duplicate resource or version conflict
- 429 Too Many Requests: Rate limit exceeded
- 500 Internal Server Error: Unhandled server errors with correlation ID
- 503 Service Unavailable: Dependency failure or circuit breaker open

**NFR-022**: The system shall implement circuit breaker pattern for all external dependencies with the following specifications:
- Circuit opens after 5 consecutive failures to external service
- Half-open state triggered after 30 seconds to test recovery
- Retry with exponential backoff (initial delay: 100ms, maximum delay: 5 seconds, maximum attempts: 3)
- Fallback responses when circuit is open: cached data where applicable, or degraded service message to user

**NFR-023**: The system shall handle database errors gracefully with the following mechanisms:
- Connection pool exhaustion: Queue requests up to 5 seconds, then return 503 Service Unavailable
- Query timeout: 10 seconds maximum execution time, log slow queries exceeding 2 seconds
- Deadlock detection: Automatic retry once, then fail with 500 Internal Server Error and alert operations

**NFR-024**: The system shall maintain automated backup strategy with the following specifications:
- Database backups: Daily full backup at 2 AM IST with 30-day retention
- File storage backups: Daily incremental backup, weekly full backup, 90-day retention
- Backup storage location: Separate AWS region from primary production region
- Recovery Time Objective (RTO): Less than 4 hours from disaster event to service restoration
- Recovery Point Objective (RPO): Less than 1 hour maximum data loss

**NFR-025**: The system shall implement high availability architecture with the following components:
- Application servers: Minimum 2 instances per availability zone across 3 availability zones (6 instances total minimum)
- Database: Master-slave replication with automatic failover completing within 60 seconds
- Load balancer health checks: Every 10 seconds, 2 consecutive failures trigger instance removal from pool
- Kubernetes readiness probes: Every 5 seconds to determine if pod can accept traffic
- Kubernetes liveness probes: Every 10 seconds to detect and restart failed pods

**NFR-026**: The system shall implement automated failover mechanisms for critical dependencies:
- Payment gateway: Automatic failover from Razorpay to PayU within 10 seconds if Razorpay returns 5xx server error
- Database: Automatic promotion of read replica to master within 60 seconds of master failure detection
- Load balancer: Automatic removal of unhealthy instances and redistribution of traffic to healthy instances

**NFR-027**: The system shall expose Prometheus-compatible metrics endpoints for all services with the following metrics:
- RED metrics: Request rate (requests/second), Error rate (errors/second), Duration (p50, p95, p99 latency)
- System metrics: CPU usage percentage, memory usage percentage, database connection pool utilization
- Business metrics: Orders per minute, gross merchandise value (GMV), vendor registrations per hour
- Alert thresholds:
  - Error rate exceeds 1% for 5 consecutive minutes: Trigger PagerDuty critical alert
  - p95 latency exceeds 3 seconds for 10 consecutive minutes: Trigger PagerDuty warning alert
  - Database connection pool exceeds 80% utilization for 5 consecutive minutes: Trigger Slack alert to operations channel
  - Disk usage exceeds 85% on any server: Trigger PagerDuty critical alert

**NFR-028**: The system shall implement structured logging with centralized aggregation meeting the following standards:
- Log format: JSON with mandatory fields (timestamp, log level, service name, trace ID, message, context object)
- Log levels: ERROR (unhandled exceptions), WARN (recoverable failures), INFO (business events), DEBUG (detailed diagnostic)
- Log retention: ERROR logs retained for 90 days, INFO logs for 30 days, DEBUG logs for 7 days
- Centralized aggregation: ELK Stack (Elasticsearch, Logstash, Kibana) for log search and analysis
- Correlation IDs: Propagate unique trace ID across all microservices for end-to-end request tracing

**NFR-029**: The system shall expose health check endpoints for all microservices with the following specifications:
- /health/liveness: Returns 200 OK if service process is running (used by Kubernetes liveness probe)
- /health/readiness: Returns 200 OK if service can handle requests with all dependencies available (used by Kubernetes readiness probe)
- /health/startup: Returns 200 OK after service initialization is complete (used by Kubernetes startup probe)
- Health check response time: Maximum 500 milliseconds
- Readiness check: Must verify connectivity to database, cache (Redis), and message queue before returning 200 OK

---

## 9. User Stories (Priority)

The following user stories represent the highest-priority customer and vendor journeys for MVP validation:

**US-001** (Must-have): As a first-time customer, I want to browse products without creating an account, so that I can evaluate the platform before committing to registration.

**US-002** (Must-have): As a customer, I want to pay using UPI on my phone, so that I can complete purchases quickly without entering card details.

**US-003** (Must-have): As a vendor, I want to see my daily sales summary on my dashboard when I log in each morning, so that I can track business performance without running a report manually.

**US-004** (Must-have): As a customer, I want to track my order in real time after it ships, so that I know when to expect delivery.

**US-005** (Should-have): As a vendor, I want to offer a discount code to my repeat customers, so that I can reward loyalty and drive repeat purchases.

**US-006** (Should-have): As a customer, I want to filter search results by price and rating, so that I can quickly find products that match my budget and quality expectations.

**US-007** (Nice-to-have): As a vendor, I want to see which of my products are most viewed but not purchased, so that I can improve listings that have high bounce rates.

---

## 10. Integration Requirements

**INT-001**: The system must integrate with the GSTN API for GST number validation during vendor onboarding.

**INT-002**: The system must integrate with Razorpay (primary) or PayU (fallback) for payment processing. The integration must support UPI, cards, net banking, and wallet payments.

**INT-003**: The system must integrate with Shiprocket API for shipment booking and tracking. The integration must support creation of shipment orders and webhook-based tracking status updates.

**INT-004**: The system must integrate with an email service provider (AWS SES or SendGrid) for transactional email delivery. Delivery rate must be monitored; emails with delivery failure rate above 2% must trigger an alert.

**INT-005**: The system must integrate with an SMS gateway (MSG91 or Twilio India) for OTP delivery and transactional SMS. OTP delivery must complete within 30 seconds of trigger.

**INT-006**: The system should integrate with Google Analytics 4 for customer behavior tracking on the storefront. Vendor dashboards should surface aggregated analytics derived from GA4 data.

---

## 11. Open Questions (Known Before Development)

The following questions must be answered before architecture phase begins:

1. **Payment split for platform campaigns (FR-054)**: What is the revenue split between ShopFlow and participating vendors for platform-wide discount campaigns? This affects payout calculation logic.

2. **Vendor tier feature differentiation**: The document references Basic (₹499/month) and Professional (₹999/month) tiers, but does not define which features are restricted to Professional. This must be defined before catalog and dashboard development begins.

3. **Return shipping responsibility**: Who bears the cost of return shipping — customer, vendor, or ShopFlow? This affects the returns flow and payout hold logic (FR-035, FR-041).

4. **COD availability criteria**: Should COD be available for all pin codes and all vendors, or should vendors be able to disable COD for their products? The business rule governing COD availability is not defined in this document.

5. **Moderation policy for reviews**: FR-051 references a "moderation policy" for prohibited review content. This policy document does not yet exist and must be defined before review system development.

6. **Vendor account suspension**: What triggers an automatic vendor account suspension (e.g., too many order cancellations, too many return complaints)? Thresholds not defined.

---

## 12. Acceptance Criteria for MVP Launch

The following conditions must all be true before the platform goes live:

- [ ] All must-have functional requirements implemented and passing QA
- [ ] OWASP Top 10 audit completed with no critical/high vulnerabilities outstanding
- [ ] WCAG 2.1 AA audit completed with no Level A or Level AA failures
- [ ] GST invoice generation validated by CA for compliance
- [ ] Payment gateway integration tested with live transactions in staging
- [ ] Load testing passing at 5,000 concurrent users
- [ ] Data residency confirmed — all production infrastructure on Indian servers
- [ ] Vendor onboarding flow tested end-to-end with 10 pilot vendors
- [ ] Consumer Protection (E-Commerce) Rules compliance checklist signed off by Legal
- [ ] Grievance Officer designated and contact published on platform

---

## 13. Glossary

| Term | Definition |
|---|---|
| Vendor | A registered business or individual selling products on the ShopFlow platform |
| Customer | An end user browsing and purchasing products on the platform |
| Storefront | A vendor's dedicated page on ShopFlow displaying their products and brand |
| GMV | Gross Merchandise Value — total value of products sold through the platform before deductions |
| Commission | ShopFlow's fee on each transaction — 8% of the order value |
| Sub-order | A portion of a customer order fulfilled by a single vendor, managed independently for tracking and payout |
| HSN code | Harmonised System of Nomenclature — a classification code used for GST rate determination |
| GSTN API | Government's API for validating GST registration numbers |
| AWB | Air Waybill — the tracking number assigned by a courier to a shipment |
| Return window | The period within which a customer can request a return — 7 days after delivery |
| Payout | The amount transferred from ShopFlow to a vendor after commission deduction |
| COD | Cash on Delivery — a payment method where the customer pays in cash when the order is delivered |
