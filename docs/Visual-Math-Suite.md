# VisualMath (AP/TG Interactive Math Suite)
## Comprehensive Application Development Action Plan & Technical Specification

**Document Version:** 1.0  
**Date:** ___________  
**Project:** VisualMath (AP/TG Interactive Math Suite)  
**Document Owner:** Project Chair / CTO  
**Classification:** Internal - Constitution Document  

---

## Document Control

| Version | Date | Author | Description of Changes |
| :--- | :--- | :--- | :--- |
| 1.0 | [Current Date] | Project Chair | Initial sign-off and comprehensive action plan creation |
| 1.1 | | | (Reserved for future Change Requests) |

### Stakeholder Sign-off Table

| Role | Name | Signature | Date |
| :--- | :--- | :--- | :--- |
| CEO | | | |
| CTO | | | |
| Product Manager | | | |
| Backend Lead | | | |
| Frontend Lead | | | |
| Database Lead | | | |
| DevOps Lead | | | |
| Legal Counsel | | | |
| Finance Lead | | | |

> **GOVERNANCE NOTICE:** This document is the constitution of the VisualMath project. Any deviation from the signed decisions herein requires a formal Change Request (CR) approved by both the CTO and CEO.

---

## Table of Contents

1. [Executive Summary & Project Mandate](#1-executive-summary--project-mandate)
2. [Project Governance & ADR Strategy](#2-project-governance--adr-strategy)
3. [Track 1: Product, Business & Legal Operations](#3-track-1-product-business--legal-operations)
    - 3.1 Annual Pricing Tier & Seat Management
    - 3.2 Trial & Grace Period Workflow
    - 3.3 Phased Rollout Strategy & Beta Cap
    - 3.4 Offline Payment Integration
    - 3.5 Government School Compliance (PFMS)
    - 3.6 Data Retention & DPDP Act Compliance
4. [Track 2: Backend Engineering & API Architecture](#4-track-2-backend-engineering--api-architecture)
    - 4.1 JWT Invalidation Mechanism
    - 4.2 Subdomain Lookup & Caching
    - 4.3 Asynchronous Tasks (Celery) & Retries
    - 4.4 API Rate Limiting Strategy
    - 4.5 Error Code Translation Architecture
    - 4.6 Structured Logging Standards
5. [Track 3: Frontend Engineering & Performance](#5-track-3-frontend-engineering--performance)
    - 5.1 State Management Approach
    - 5.2 Bundle Size Budgets & Code Splitting
    - 5.3 Heavy Library Loading (Three.js/D3)
    - 5.4 Browser Support Matrix
    - 5.5 Offline Resilience (PWA & Workbox)
    - 5.6 FPS Degradation Strategy
    - 5.7 Memory Cleanup Enforcement
6. [Track 4: Database Engineering & Storage](#6-track-4-database-engineering--storage)
    - 6.1 Primary Key Strategy
    - 6.2 Table Partitioning (TeacherActivity)
    - 6.3 Connection Pooling (PgBouncer)
    - 6.4 Backup Strategy (RPO/RTO)
    - 6.5 Autovacuum Tuning
    - 6.6 Migration Safety Rules
7. [Track 5: DevOps, Infrastructure & Cloud Cost Management](#7-track-5-devops-infrastructure--cloud-cost-management)
    - 7.1 SSL Certificate Strategy
    - 7.2 Reverse Proxy Configuration
    - 7.3 CI/CD Pipeline & OpenAPI Contract Enforcement
    - 7.4 Observability & APM
    - 7.5 Cloud Cost Alerting & Kill Switch
    - 7.6 Disaster Recovery & Chaos Engineering
    - 7.7 CDN Cache Invalidation
8. [Track 6: Cross-Team Integration & QA](#8-track-6-cross-team-integration--qa)
    - 8.1 Staging Environment Parity
    - 8.2 On-Call Escalation Matrix
    - 8.3 Feature Flag Strategy
    - 8.4 Bilingual Error Reporting
9. [Implementation Timeline & Milestones](#9-implementation-timeline--milestones)
10. [Risk Management Matrix](#10-risk-management-matrix)
11. [Appendix A: Architecture Decision Records (ADR) Template](#11-appendix-a-architecture-decision-records-adr-template)
12. [Appendix B: API Error Code Schema Specification](#12-appendix-b-api-error-code-schema-specification)

---

## 1. Executive Summary & Project Mandate

The VisualMath (AP/TG Interactive Math Suite) project aims to deliver a highly interactive, 3D/2D math simulation platform capable of serving up to 1 million schools. The platform must perform reliably on constrained school hardware and intermittent internet connections. 

Following a comprehensive cross-functional alignment meeting, the core leadership team has ratified the "FINAL ACTION ITEMS" as the technological and business constitution of the project. This document translates those signed decisions into an exhaustive, actionable engineering and business plan.

### Core Mandates:
1. **Financial Prudence:** Lock pricing tiers and implement strict cloud cost-alert thresholds to prevent billing shocks.
2. **Performance at Scale:** Commit to monthly database partitioning and aggressive CDN caching to handle massive concurrency.
3. **Reliability & Safety:** Mandate monthly "Chaos Drills" to test Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO).
4. **Team Safety:** Implement a CI/CD hard gate for OpenAPI contracts to prevent backend/frontend integration surprises.

---

## 2. Project Governance & ADR Strategy

To ensure the project remains aligned with the initial architectural decisions, strict governance must be enforced.

### 2.1 Architecture Decision Records (ADRs)
All decisions made in the sign-off meeting must be codified into the codebase.
*   **Location:** `docs/decisions/`
*   **Format:** Markdown files named `NNNN-short-title.md` (e.g., `0001-subdomain-caching-redis.md`).
*   **Content:** Context, Decision, Status, Consequences.

### 2.2 Change Request (CR) Process
Any engineer or lead who identifies a need to deviate from this document must:
1. Draft a CR detailing the current signed decision, the proposed deviation, and the technical/business justification.
2. Submit the CR for review.
3. **Hard Gate:** No implementation of the deviation may begin until formal approval is granted by both the CTO and CEO.

### 2.3 Follow-up Cadence
*   **Weekly:** Engineering syncs to track progress on critical path items (Subdomain Middleware, Partitioning Script, CI/CD Pipeline).
*   **Bi-Weekly:** Cross-functional lead review to ensure domain tracks are not blocking each other.

---

## 3. Track 1: Product, Business & Legal Operations

**Team:** CEO, Sales, Product Manager, Legal Counsel, Finance  
**Objective:** Define monetization, market rollout, and statutory compliance.

### 3.1 Annual Pricing Tier & Seat Management
**Decision Reference:** 1.1  
**Owner:** CEO / Sales / Product

The billing system must strictly enforce tier limits. User interface must prevent administrators from inviting teachers beyond their tier capacity.

**Action Items:**
*   **Backend Implementation:** 
    *   Create a `Subscription` model linked to `School`.
    *   Fields: `tier` (BASIC, PREMIUM), `max_teacher_seats` (5 or 15), `renewal_date`.
    *   Implement a database trigger or Django signal that prevents `TeacherProfile` creation if `school.teachers.count() >= school.subscription.max_teacher_seats`.
*   **Frontend Implementation:**
    *   Display seat utilization in the Admin Dashboard (e.g., "3 / 5 seats used").
    *   Disable the "Invite Teacher" button when capacity is reached, providing an upsell prompt to upgrade to Premium.
*   **Acceptance Criteria:** QA must verify that attempting to add a 6th teacher on a Basic tier returns a `402 Payment Required` or `403 Forbidden` with error code `E_SEAT_LIMIT_REACHED`.

### 3.2 Trial & Grace Period Workflow
**Decision Reference:** 1.2  
**Owner:** CEO / Product

The system must allow schools to evaluate the product before committing financial resources, while automatically degrading service if payment is delayed.

**Action Items:**
*   **State Machine:** Design a subscription state machine: `TRIAL_ACTIVE` -> `TRIAL_EXPIRED` -> `GRACE_PERIOD` -> `ACTIVE` -> `SUSPENDED`.
*   **Trial Phase:** Upon signup, `TRIAL_ACTIVE` is set with an expiry date based on the agreed duration (7 or 14 days).
*   **Grace Phase:** Upon trial expiry, if no payment is processed, status moves to `GRACE_PERIOD` (7 or 14 days). 
    *   *Read-Only Enforcement:* Backend middleware must intercept all `POST`, `PUT`, `PATCH`, `DELETE` requests for schools in `GRACE_PERIOD` and return `403 Forbidden` with `E_GRACE_MODE_READ_ONLY`. GET requests remain unaffected.
*   **Suspension:** After the grace period expires, status moves to `SUSPENDED`. All API requests return `403` with `E_SUBSCRIPTION_EXPIRED`.

### 3.3 Phased Rollout Strategy & Beta Cap
**Decision Reference:** 1.3  
**Owner:** Product / Ops

To mitigate risk, the rollout will be phased (Option B: 50-School Pilot in Hyderabad/Vizag).

**Action Items:**
*   **Geofencing/Domain Whitelisting:** Configure the backend to accept signups only from specific school domains or IP ranges associated with the 50 pilot schools.
*   **Daily Signup Cap:** Implement a Redis counter (`beta_signups_YYYYMMDD`) to enforce the maximum daily signup limit. If the cap is hit, the registration endpoint returns `429 Too Many Requests` with `E_BETA_CAP_REACHED`.
*   **Ops Readiness:** Customer support must be staffed to handle the expected volume from the 50 pilot schools.

### 3.4 Offline Payment Integration
**Decision Reference:** 1.4  
**Owner:** Finance / CTO

Depending on the signed decision, manual offline payment workflows may be required for institutions that cannot use RazorPay.

**Action Items (If "Yes" selected):**
*   **Django Admin UI:** Create a custom admin page for Finance to input Purchase Order (PO) numbers or Cheque references.
*   **Manual Activation:** Finance clicks "Activate Subscription," which bypasses the payment gateway and sets the subscription status to `ACTIVE` with a future `renewal_date`.
*   **Audit Log:** All manual activations must be logged with the admin user ID, timestamp, and PO/Cheque number for financial auditing.

### 3.5 Government School Compliance (PFMS)
**Decision Reference:** 1.5  
**Owner:** Legal / Sales

**Action Items (If "Avoid PFMS" selected):**
*   Add an exclusion criteria in the signup form. If a user selects "Government School" as the institution type, display a message: "We currently only serve private institutions. Government school support coming soon."
*   Redirect government school inquiries to a waitlist form for future outreach.

**Action Items (If "Integrate PFMS" selected):**
*   Initiate a parallel 6-month sub-project.
*   Assign a dedicated API integration specialist to liaise with PFMS administrators.
*   Ensure the database schema accommodates PFMS-specific transaction IDs and sanction codes.

### 3.6 Data Retention & DPDP Act Compliance
**Decision Reference:** 1.6  
**Owner:** Legal / CTO

The Digital Personal Data Protection (DPDP) Act requires strict handling of user data.

**Action Items:**
*   **Cron Job Development:** Create a Celery Beat task running daily at 02:00 UTC.
*   **Logic:**
    *   Query `TeacherProfile` where `last_login_date < NOW() - INTERVAL '3 years'` (or 5 years, based on decision).
    *   *Anonymization Strategy:* Replace `first_name`, `last_name`, `email`, `phone_number` with hashed or null values. Retain the profile ID only for historical aggregate analytics.
    *   *Deletion Strategy (If chosen):* Completely delete the record and cascade delete activity logs.
*   **Legal Review:** Legal counsel must sign off on the exact anonymization script before it is deployed to production.

---

## 4. Track 2: Backend Engineering & API Architecture

**Team:** Backend Lead, Backend Developers  
**Objective:** Build a secure, multi-tenant, high-performance API architecture using Django.

### 4.1 JWT Invalidation Mechanism
**Decision Reference:** 2.1  
**Owner:** Backend Lead

Handling JSON Web Tokens (JWT) securely is critical for session management.

**Action Items:**
*   **If Stateful (Option A):**
    *   Include a `last_password_change` timestamp in the User model.
    *   Include the `iat` (issued at) claim in the JWT payload.
    *   **Middleware:** Write custom Django middleware that intercepts every request, extracts the JWT, and compares `jwt.iat` with `user.last_password_change`. If `jwt.iat < user.last_password_change`, reject the request with `401 Unauthorized` and code `E_TOKEN_EXPIRED`. Use Redis to cache `last_password_change` to avoid DB hits on every request.
*   **If Stateless (Option B):**
    *   Configure the JWT library (e.g., `djangorestframework-simplejwt`) to enforce a strict short Access Token TTL (6 hours).
    *   Rely entirely on the refresh token mechanism. No DB lookups for token validation.

### 4.2 Subdomain Lookup & Caching
**Decision Reference:** 2.2  
**Owner:** Backend Lead

The application uses wildcard subdomains (e.g., `schoolname.visualmath.in`). Every API request must resolve the subdomain to a `School` ID efficiently.

**Action Items:**
*   **Middleware Creation:** `SubdomainTenantMiddleware`.
*   **Logic:**
    1. Extract subdomain string from `request.META['HTTP_HOST']`.
    2. Check Redis using key `tenant:subdomain:{subdomain_str}`.
    3. If Redis hit: Attach `school_id` to `request.school_id` and proceed.
    4. If Redis miss: Query `School.objects.get(subdomain=subdomain_str)`.
    5. Cache the result in Redis with a 24-hour TTL.
    6. If DB miss: Return `404 Not Found`.
*   **Cache Invalidation:** When a school's subdomain is updated in Django Admin, dispatch a signal to delete the specific Redis key.

### 4.3 Asynchronous Tasks (Celery) & Retries
**Decision Reference:** 2.3  
**Owner:** Backend Lead

Background jobs (e.g., sending emails, generating reports) require robust retry mechanisms.

**Action Items:**
*   Configure Celery with RabbitMQ or Redis as the broker.
*   Define a base Task class with the agreed retry policy.
    ```python
    class BaseRetryTask(celery.Task):
        autoretry_for = (Exception,)
        max_retries = 5 # Based on signed decision
        retry_backoff = True
        retry_backoff_max = 600
        retry_jitter = True
    ```
*   Apply this base class to all critical background tasks.

### 4.4 API Rate Limiting Strategy
**Decision Reference:** 2.4  
**Owner:** Backend Lead

Simulations are computationally heavy. Rate limiting protects backend resources.

**Action Items:**
*   Utilize `django-ratelimit` or DRF's built-in throttling.
*   **Implementation (Per School example):**
    *   Scope: `simulations_run`.
    *   Rate: `100/min` (based on signed decision).
    *   Key function: `lambda g: g.request.school_id`.
*   **Response:** When limit is exceeded, return `429 Too Many Requests` and include a `Retry-After` header.

### 4.5 Error Code Translation Architecture
**Decision Reference:** 2.5  
**Owner:** Backend + FE Lead

To decouple backend messaging from frontend UI, the backend will only return error codes.

**Action Items:**
*   **Backend:** Standardize all API error responses to the following JSON structure:
    ```json
    {
      "error": {
        "code": "E_SUB_EXPIRED",
        "details": {} 
      }
    }
    ```
*   **Frontend:** Fetch a `errors.json` file from the CDN. Map `E_SUB_EXPIRED` to the appropriate bilingual string. (See Track 6 for bilingual implementation).

### 4.6 Structured Logging Standards
**Decision Reference:** 2.6  
**Owner:** DevOps / Backend Lead

Logs must be machine-readable for APM integration.

**Action Items:**
*   Configure Python `structlog` or Django's logging formatter to output JSON.
*   **Required Fields:** `timestamp`, `level`, `logger_name`, `message`, `request_id`, `school_id`, `user_id`.
*   Ensure `request_id` is generated by middleware and passed through to Celery tasks to trace requests across the stack.

---

## 5. Track 3: Frontend Engineering & Performance

**Team:** Frontend Lead, Frontend Developers, UI/UX Designers  
**Objective:** Deliver a highly interactive, smooth 3D/2D math experience that works on constrained school hardware.

### 5.1 State Management Approach
**Decision Reference:** 3.1  
**Owner:** FE Lead

**Action Items:**
*   Avoid heavy state management libraries like Redux if bundle size is a concern. Implement Custom Pub/Sub using `mitt`.
*   Create a `EventBus.js` utility:
    ```javascript
    import mitt from 'mitt'
    const emitter = mitt()
    export default emitter
    ```
*   Use this bus for cross-component communication (e.g., theme changes, global notifications) instead of prop-drilling or global `window` variables.

### 5.2 Bundle Size Budgets & Code Splitting
**Decision Reference:** 3.2  
**Owner:** FE Lead

**Action Items:**
*   Configure Vite or Webpack performance hints to error out if the initial bundle exceeds the hard budget (e.g., 150 KB gzipped).
    ```javascript
    // vite.config.js
    build: {
      rollupOptions: {
        output: {
          manualChunks: {
            vendor: ['react', 'react-dom'],
          }
        }
      },
      chunkSizeWarningLimit: 150,
    }
    ```
*   CI pipeline must fail if the `dist` folder's total initial load size exceeds the budget.

### 5.3 Heavy Library Loading (Three.js/D3)
**Decision Reference:** 3.3  
**Owner:** FE Lead

**Action Items:**
*   Never import `three` or `d3` statically at the top of the file.
*   Use dynamic imports inside the component's `useEffect` or mounted hook:
    ```javascript
    useEffect(() => {
      let cleanup;
      import('three').then((THREE) => {
        // Initialize 3D scene
        cleanup = initScene(THREE);
      });
      return () => cleanup && cleanup();
    }, []);
    ```

### 5.4 Browser Support Matrix
**Decision Reference:** 3.4  
**Owner:** FE Lead

**Action Items:**
*   Configure `browserslist` in `package.json`:
    ```json
    "browserslist": [
      "Chrome >= 88",
      "Edge >= 88",
      "Firefox >= 90",
      "not IE11"
    ]
    ```
*   Add a routing guard that checks `navigator.userAgent`. If IE11 is detected, render a blocking overlay: "VisualMath requires a modern browser (Chrome 88+). Please update your browser."

### 5.5 Offline Resilience (PWA & Workbox)
**Decision Reference:** 3.5  
**Owner:** FE Lead

**Action Items:**
*   Implement a Service Worker using Workbox.
*   **Strategy:**
    *   App Shell: Precache the core HTML/CSS/JS.
    *   API GET Requests: Use `NetworkFirst` strategy (tries network, falls back to cache).
    *   API POST/PUT Requests: Implement a background sync queue. If offline, store the request in IndexedDB. When connection restores, replay the queue.

### 5.6 FPS Degradation Strategy
**Decision Reference:** 3.6  
**Owner:** FE Lead

**Action Items:**
*   Write a custom hook `useFPSMonitor` that calculates frames per second.
*   If FPS < 30 for a continuous 5-second window, dispatch an event to the 3D renderer.
*   **Renderer Response:** Reduce particle count, lower texture resolution, or disable complex shaders dynamically to stabilize performance.

### 5.7 Memory Cleanup Enforcement
**Decision Reference:** 3.7  
**Owner:** FE Lead

**Action Items:**
*   Memory leaks in 3D apps are catastrophic. Mandate that every heavy component must export a `cleanup()` function.
*   **ESLint Rule:** Write a custom ESLint rule or enforce via PR review that components interacting with WebGL/Canvas call `renderer.dispose()` and remove event listeners in a `useEffect` cleanup return.

---

## 6. Track 4: Database Engineering & Storage

**Team:** DB Lead, Data Engineers  
**Objective:** Ensure data integrity, fast query times, and safe schema evolution for 1M+ schools.

### 6.1 Primary Key Strategy
**Decision Reference:** 4.1  
**Owner:** DB Lead

**Action Items:**
*   If UUID v7: Ensure all models use `UUIDField(primary_key=True, default=uuid7)`. UUID v7 is time-ordered, which helps B-Tree index performance compared to v4.
*   If BIGINT: Standard Django `BigAutoField`. Ensure sequence caching is configured in PostgreSQL to avoid bottlenecks on high-volume inserts.

### 6.2 Table Partitioning (TeacherActivity)
**Decision Reference:** 4.2  
**Owner:** DB Lead

`TeacherActivity` will become a massive table. Partitioning is mandatory.

**Action Items:**
*   Use `pg_partman` or a custom Django management command.
*   **SQL Structure:**
    ```sql
    CREATE TABLE teacher_activity (
        id UUID,
        school_id UUID,
        teacher_id UUID,
        action VARCHAR,
        created_at TIMESTAMP
    ) PARTITION BY RANGE (created_at);
    ```
*   **Automation:** Create a Celery Beat task that runs on the 1st of every month, creating the partition for the month 3-6 months in the future. (e.g., On Jan 1st, create the partition for April).

### 6.3 Connection Pooling (PgBouncer)
**Decision Reference:** 4.3  
**Owner:** DB Lead

**Action Items:**
*   Deploy PgBouncer between Django and PostgreSQL.
*   Configure `pool_mode = transaction`.
*   Update Django `DATABASES` setting: Set `'DISABLE_PREPARED_STATEMENTS': True` to prevent issues with transaction pooling.
*   Set pool size based on max DB connections.

### 6.4 Backup Strategy (RPO/RTO)
**Decision Reference:** 4.4  
**Owner:** DB Lead / DevOps

**Action Items:**
*   **RPO (Recovery Point Objective):** Configure continuous WAL archiving or automated snapshots every 5 or 15 minutes.
*   **RTO (Recovery Time Objective):** Document and test the restoration procedure to guarantee the database is back online within 15 or 30 minutes.
*   Store encrypted backups in a different availability zone or region.

### 6.5 Autovacuum Tuning
**Decision Reference:** 4.5  
**Owner:** DB Lead

**Action Items:**
*   Modify `postgresql.conf`:
    *   If Aggressive: `autovacuum_vacuum_scale_factor = 0.01` (Triggers vacuum when 1% of rows change).
    *   Ensure `autovacuum_max_workers` is sufficient for the number of partitions.

### 6.6 Migration Safety Rules
**Decision Reference:** 4.6  
**Owner:** CTO / DB Lead

**Action Items:**
*   Create a CONTRIBUTING.md rule: No direct `ALTER TABLE` on large tables.
*   **3-Step Rule Process:**
    1. **Add:** Create the new column nullable, or create the new table.
    2. **Backfill:** Write a Django data migration script to populate the data in batches.
    3. **Drop:** After verifying the backfill, drop the old column.
*   CI pipeline must flag migrations that use `AlterField` on known large tables for manual DBA review.

---

## 7. Track 5: DevOps, Infrastructure & Cloud Cost Management

**Team:** DevOps Lead, CTO, SRE  
**Objective:** Build a cost-controlled, observable, and highly available cloud environment.

### 7.1 SSL Certificate Strategy
**Decision Reference:** 5.1  
**Owner:** DevOps

**Action Items:**
*   Procure a Wildcard SSL certificate for `*.visualmath.in` to avoid Let's Encrypt rate limits (which can occur if spawning hundreds of subdomains rapidly).
*   Alternatively, rely on Caddy's built-in On-Demand TLS, but configure it with an ask endpoint to verify subdomain validity before issuing certs.

### 7.2 Reverse Proxy Configuration
**Decision Reference:** 5.2  
**Owner:** DevOps

**Action Items:**
*   Deploy Caddy as the ingress controller/reverse proxy.
*   **Caddyfile Example:**
    ```caddyfile
    *.visualmath.in {
        reverse_proxy localhost:8000 # Django Backend
        tls {
            ask http://localhost:8000/api/v1/check-subdomain
        }
    }
    ```

### 7.3 CI/CD Pipeline & OpenAPI Contract Enforcement
**Decision Reference:** 5.3  
**Owner:** Backend + FE Lead

**Action Items:**
*   Backend generates `openapi.yaml` via DRF Spectacular on every commit.
*   Frontend generates types or validates against this schema.
*   **CI Gate:** Create a GitHub Action (or GitLab CI step) that runs `openapi-diff` between the PR branch and `main`. If breaking changes are detected without frontend alignment, the pipeline fails hard.

### 7.4 Observability & APM
**Decision Reference:** 5.4  
**Owner:** DevOps / CTO

**Action Items:**
*   Deploy chosen APM agent (Datadog, NewRelic, or Prometheus/Grafana stack).
*   Ensure Django middleware traces are enabled.
*   Create dashboards for: API Latency (p99), Error Rates, Celery Queue Backlog, DB CPU, and Cache Hit Ratio.

### 7.5 Cloud Cost Alerting & Kill Switch
**Decision Reference:** 5.5  
**Owner:** DevOps / CFO

**Action Items:**
*   Configure AWS Budgets or GCP Billing Alerts at 80% of the monthly threshold.
*   **Kill Switch:** Create a Lambda/Cloud Function that triggers at the kill switch threshold %.
    *   *Action:* Auto-throttle the auto-scaling group maximum size to 1, or shut down non-critical worker nodes, preventing a massive bill spike. Notify the CTO immediately.

### 7.6 Disaster Recovery & Chaos Engineering
**Decision Reference:** 5.6  
**Owner:** DevOps Lead

**Action Items:**
*   **Chaos Monday:** On the first Monday of every month, the DevOps team will simulate a failure in Staging (and occasionally Production, if highly resilient).
*   **Scenarios:** Kill the primary DB instance, delete a Redis node, or block network access to the CDN.
*   **Goal:** Verify automated failover triggers, RTO is met, and alerts fire correctly. Write a post-mortem for any failures.

### 7.7 CDN Cache Invalidation
**Decision Reference:** 5.7  
**Owner:** DevOps

**Action Items:**
*   Add a post-deploy hook in the CI/CD pipeline.
*   After the new frontend build is deployed to S3/CloudFront, automatically trigger an invalidation for `/*` or specific asset paths to ensure users get the new code immediately.

---

## 8. Track 6: Cross-Team Integration & QA

**Team:** QA Leads, CTO, All Leads  
**Objective:** Ensure smooth handoffs between teams and reliable incident response.

### 8.1 Staging Environment Parity
**Decision Reference:** 6.1  
**Owner:** DevOps

**Action Items:**
*   Staging must use the same Docker images, OS version, and PostgreSQL version as Production.
*   Never use SQLite in Staging. Staging DB should be a sanitized clone of Production data weekly to ensure realistic query plans.

### 8.2 On-Call Escalation Matrix
**Decision Reference:** 6.2  
**Owner:** CTO

**Action Items:**
*   Input the agreed schedule into PagerDuty or Opsgenie.
*   **Escalation Policy:** If P1 alert is not acknowledged in 5 mins, escalate to CTO.
*   Ensure all on-call engineers have access to the runbook repository.

### 8.3 Feature Flag Strategy
**Decision Reference:** 6.3  
**Owner:** Backend Lead

**Action Items:**
*   Create a `FeatureFlag` model: `key`, `is_active`, `allowed_schools` (Many-to-Many).
*   Wrap new features in a decorator or middleware:
    ```python
    if not FeatureFlag.is_enabled('new_3d_graphing', request.school):
        return Response({"error": {"code": "E_FEATURE_DISABLED"}}, status=403)
    ```

### 8.4 Bilingual Error Reporting
**Decision Reference:** 6.4  
**Owner:** Product / FE Lead

**Action Items:**
*   Create a localization JSON file served via CDN.
*   Implement a language toggle in the UI.
*   When an error code (e.g., `E_SUB_EXPIRED`) is received, the frontend looks up the translation based on the user's selected language (English or Telugu).

---

## 9. Implementation Timeline & Milestones

| Week | Track | Milestone | Dependency |
| :--- | :--- | :--- | :--- |
| 1 | All | Phase 0: ADRs created, CR process established | None |
| 2 | DevOps | Caddy configured, Wildcard SSL active | Domain procurement |
| 2 | Backend | Subdomain Middleware (Redis) deployed to Staging | DevOps infra |
| 3 | Database | Monthly Partitioning Script tested | DB provisioning |
| 3 | DevOps | CI/CD OpenAPI Hard Gate operational | BE/FE schema definition |
| 4 | Backend | JWT Invalidation & Rate Limiting implemented | Subdomain Middleware |
| 5 | Frontend| Bundle budgets enforced, Dynamic imports refactored | None |
| 6 | Frontend| Workbox Offline queue implemented | BE API stability |
| 6 | Database| PgBouncer transaction pooling active | DB tuning |
| 7 | Backend | Celery Retries & Error Codes standardized | None |
| 8 | Frontend| FPS Degradation & Cleanup CI rules active | 3D component completion |
| 9 | Legal | DPDP Data Retention script tested | BE Cron jobs |
| 10| Product| Trial/Grace Period state machine tested | BE Billing logic |
| 11| DevOps | First "Chaos Monday" DR Drill executed | Full infra deployment |
| 12| All | 50-School Pilot Launch (Beta) | All tracks complete |

---

## 10. Risk Management Matrix

| Risk ID | Description | Probability | Impact | Mitigation Strategy | Owner |
| :--- | :--- | :--- | :--- | :--- | :--- |
| R-01 | Cloud cost spike due to simulation abuse | Medium | High | API Rate limiting (4.4) and Auto-throttle Kill Switch (5.5) | DevOps |
| R-02 | DB bloat causing query timeouts | High | High | Monthly Partitioning (4.2) and Aggressive Autovacuum (4.5) | DB Lead |
| R-03 | Frontend crashes on low-end school PCs | High | High | FPS Degradation (3.6), Bundle Budgets (3.2), Dynamic imports (3.3) | FE Lead |
| R-04 | Integration block between BE and FE | Medium | Critical | CI/CD OpenAPI Hard Gate (5.3) | CTO |
| R-05 | Data loss during outage | Low | Critical | RPO 5-min backups (4.4), Monthly Chaos Drills (5.6) | DevOps |
| R-06 | Legal action due to DPDP non-compliance | Low | High | Automated Anonymization script (3.6) | Legal |

---

## 11. Appendix A: Architecture Decision Records (ADR) Template

```markdown
# ADR [Number]: [Title]

## Status
[Proposed | Accepted | Deprecated | Superseded]

## Context
[Why is this decision required? What are the technical/business drivers?]

## Decision
[What is the exact change or implementation choice we are making?]

## Consequences
[What are the pros, cons, and impacts of this decision? E.g., performance gains, increased operational complexity.]
```

---

## 12. Appendix B: API Error Code Schema Specification

The backend will exclusively return the following JSON structure for all non-2xx responses. The Frontend will map the `code` to a localized UI string.

```json
{
  "error": {
    "code": "E_XXXX_YYYY",
    "field": "field_name", // Optional: present only for validation errors
    "details": {
      // Optional: additional context
    }
  }
}
```

### Standardized Error Codes (Initial Set)

| Code | HTTP Status | Description | Frontend Action |
| :--- | :--- | :--- | :--- |
| `E_AUTH_INVALID` | 401 | Invalid credentials | Redirect to login |
| `E_TOKEN_EXPIRED` | 401 | JWT is expired | Attempt silent refresh |
| `E_SEAT_LIMIT_REACHED`| 403 | School has max teachers | Show upgrade prompt |
| `E_GRACE_MODE_READ_ONLY`| 403 | Trial expired, in grace period | Show "Upgrade to Edit" banner |
| `E_SUBSCRIPTION_EXPIRED`| 403 | Subscription suspended | Redirect to billing page |
| `E_BETA_CAP_REACHED` | 429 | Daily signup limit hit | Show "Try again tomorrow" msg |
| `E_FEATURE_DISABLED` | 403 | Feature flag off for school | Hide UI element |
| `E_RATE_LIMITED` | 429 | Too many simulation requests | Show "Slow down" toast |
| `E_SUBDOMAIN_NOT_FOUND`| 404 | School does not exist | Redirect to main landing page |
