## FINAL ACTION ITEMS: THE "MUST-AGREE" SIGN-OFF SHEET

**Meeting Date:** ___________  
**Project:** MathVision (AP/TG Interactive Math Suite)  
**Chair:** ___________  

---

### DOMAIN 1: PRODUCT, PRICING & ROLLOUT (Stakeholders / CEO / Sales)

| # | Decision Point | Options Presented | FINAL SIGNED DECISION *(Choose & Fill)* | Owner |
| :--- | :--- | :--- | :--- | :--- |
| 1.1 | **Annual Pricing Tier** | (A) ₹5,000/year (Basic) <br> (B) ₹10,000/year (Basic) <br> (C) ₹25,000/year (Premium with 15 teachers) | **Choice:** _______ <br> **Teacher Seats per Tier:** Basic = 5 / Premium = 15 | CEO / Sales |
| 1.2 | **Trial & Grace Period** | (A) 7-day Trial + 7-day Grace (Read-only) <br> (B) 14-day Trial + 14-day Grace <br> (C) No Trial (Pay upfront) | **Trial:** _______ days <br> **Grace (Read-only):** _______ days | CEO / Product |
| 1.3 | **Rollout Strategy (Phasing)** | (A) Direct Public Launch (All districts) <br> (B) 50-School Pilot (Hyderabad/Vizag) for 3 months <br> (C) District-by-District rollout | **Choice:** _______ <br> **Beta Cap (Max Daily Signups):** _______ | Product / Ops |
| 1.4 | **Offline Payment (PO/Cheque)** | (A) Yes—Build manual activation UI in Django Admin <br> (B) No—RazorPay only (strictly online) | **Choice:** _______ | Finance / CTO |
| 1.5 | **Government School Compliance** | (A) Integrate with PFMS (6-month project) <br> (B) Avoid PFMS; target only Private Schools initially | **Choice:** _______ | Legal / Sales |
| 1.6 | **Data Retention (DPDP Act)** | (A) Anonymize teacher data after 3 years <br> (B) Anonymize after 5 years <br> (C) Delete permanently after expiry | **Choice:** _______ | Legal / CTO |

---

### DOMAIN 2: BACKEND ARCHITECTURE & SECURITY (CTO / Backend Lead)

| # | Decision Point | Options Presented | FINAL SIGNED DECISION | Owner |
| :--- | :--- | :--- | :--- | :--- |
| 2.1 | **JWT Invalidation Mechanism** | (A) Stateful: Check `last_password_change` on every request (DB/REDIS hit) <br> (B) Stateless: Rely solely on short 6-hour expiry | **Choice:** _______ <br> **Access Token TTL:** _______ hours | Backend Lead |
| 2.2 | **Subdomain Lookup Caching** | (A) Redis Cache (24hr TTL) with DB fallback <br> (B) Direct DB query every time (No cache) | **Choice:** _______ | Backend Lead |
| 2.3 | **Celery Retry Policy (Max Attempts)** | (A) 3 retries <br> (B) 5 retries <br> (C) 10 retries | **Max Retries:** _______ <br> **Exponential Backoff Base (seconds):** _______ | Backend Lead |
| 2.4 | **API Rate Limiting (Simulations)** | (A) 100 req/min per School <br> (B) 200 req/min per School <br> (C) 50 req/min per Teacher | **Limit:** _______ req/min per _______ | Backend Lead |
| 2.5 | **Error Code Strategy** | (A) Backend returns full error messages (hardcoded) <br> (B) Backend returns codes (e.g., E_SUB_EXPIRED); FE translates via CDN JSON | **Choice:** _______ | Backend + FE Lead |
| 2.6 | **Structured Logging** | (A) JSON logs (for Datadog/CloudWatch) <br> (B) Plain text logs | **Choice:** _______ | DevOps |

---

### DOMAIN 3: FRONTEND PERFORMANCE & HARDWARE (Frontend Lead)

| # | Decision Point | Options Presented | FINAL SIGNED DECISION | Owner |
| :--- | :--- | :--- | :--- | :--- |
| 3.1 | **State Management Approach** | (A) Custom Pub/Sub (EventEmitter / `mitt`) <br> (B) Global `window` variables (risky) | **Choice:** _______ | FE Lead |
| 3.2 | **Main Bundle Size Budget** | (A) ≤ 150 KB gzipped <br> (B) ≤ 300 KB gzipped <br> (C) No budget (risk) | **Hard Budget:** _______ KB | FE Lead |
| 3.3 | **Heavy Library Loading (Three.js/D3)** | (A) Dynamic ES6 `import()` on-demand <br> (B) Load everything at login | **Choice:** _______ | FE Lead |
| 3.4 | **Browser Support Cutoff** | (A) Chrome 88+, Edge 88+, Firefox 90+ <br> (B) Chrome 80+, Firefox 85+ <br> (C) IE11 (NO—Reject) | **Minimum Chrome Version:** _______ <br> **Block IE11?** Yes / No | FE Lead |
| 3.5 | **Offline Resilience** | (A) Build Service Worker + IndexedDB queue (Workbox) <br> (B) No offline support (rely on network) | **Choice:** _______ | FE Lead |
| 3.6 | **FPS Degradation Strategy** | (A) Auto-downscale renderer if FPS < 30 for 5s <br> (B) Manual toggle only (no auto) | **Choice:** _______ | FE Lead |
| 3.7 | **Memory Cleanup Enforcement** | (A) Mandatory `cleanup()` export with CI lint check <br> (B) No enforcement (developer responsibility) | **Choice:** _______ | FE Lead |

---

### DOMAIN 4: DATABASE & STORAGE (DB Lead / DevOps)

| # | Decision Point | Options Presented | FINAL SIGNED DECISION | Owner |
| :--- | :--- | :--- | :--- | :--- |
| 4.1 | **Primary Key Type** | (A) UUID v7 (time-ordered) <br> (B) Auto-increment `BIGINT` (SERIAL) | **Choice:** _______ | DB Lead |
| 4.2 | **`TeacherActivity` Partitioning** | (A) Monthly RANGE partitions <br> (B) Yearly RANGE partitions <br> (C) No partitioning (risk) | **Choice:** _______ <br> **Partition Creation Lead Time:** _______ months in advance | DB Lead |
| 4.3 | **PgBouncer Pooling Mode** | (A) Transaction Pooling + `DISABLE_PREPARED_STATEMENTS` <br> (B) Session Pooling | **Choice:** _______ <br> **Pool Size:** _______ | DB Lead |
| 4.4 | **Backup RPO / RTO** | RPO: (A) 5 mins / (B) 15 mins <br> RTO: (A) 15 mins / (B) 30 mins | **RPO:** _______ mins <br> **RTO:** _______ mins | DB Lead / DevOps |
| 4.5 | **Autovacuum Tuning** | (A) Aggressive: `scale_factor=0.01` <br> (B) Default: `scale_factor=0.2` | **Choice:** _______ | DB Lead |
| 4.6 | **Migration Safety Rule** | (A) Enforce "Add → Backfill → Drop" 3-step rule (CI review) <br> (B) Allow direct `ALTER TABLE` (risk) | **Choice:** _______ | CTO / DB Lead |

---

### DOMAIN 5: DEVOPS, INFRASTRUCTURE & COST (DevOps / CTO)

| # | Decision Point | Options Presented | FINAL SIGNED DECISION | Owner |
| :--- | :--- | :--- | :--- | :--- |
| 5.1 | **SSL Certificate Strategy** | (A) Wildcard `*.mathvision.com` (Caddy handles auto-renew) <br> (B) Individual certs per subdomain (Let's Encrypt rate-limit risk) | **Choice:** _______ | DevOps |
| 5.2 | **Reverse Proxy** | (A) Caddy (auto-SSL, simpler config) <br> (B) Nginx + Lua scripting (complex) | **Choice:** _______ | DevOps |
| 5.3 | **CI/CD Gate - OpenAPI Contract** | (A) Hard Fail: Pipeline breaks if FE/BE schemas mismatch <br> (B) Warning only (advisory) | **Choice:** _______ | Backend + FE Lead |
| 5.4 | **Observability / APM Tool** | (A) DataDog (expensive, comprehensive) <br> (B) NewRelic <br> (C) Open-source self-hosted (Prometheus/Grafana) | **Choice:** _______ | DevOps / CTO |
| 5.5 | **Monthly Cost Alert Threshold** | Alert at (A) 80% / (B) 90% of budget <br> **Kill Switch:** Auto-throttle if costs exceed _______ % | **Alert:** _______ % <br> **Kill Switch Threshold:** _______ % | DevOps / CFO |
| 5.6 | **Disaster Recovery Drill Schedule** | (A) Monthly (Chaos Monday) <br> (B) Quarterly <br> (C) Never (high risk) | **Frequency:** _______ | DevOps Lead |
| 5.7 | **CloudFront Cache Invalidation** | (A) Auto-invalidate on every production deploy <br> (B) Manual invalidation only | **Choice:** _______ | DevOps |

---

### DOMAIN 6: CROSS-TEAM INTEGRATION (ALL LEADS)

| # | Decision Point | Options Presented | FINAL SIGNED DECISION | Owner |
| :--- | :--- | :--- | :--- | :--- |
| 6.1 | **Staging Environment Parity** | (A) Mirror Prod OS/Dependencies (same, smaller hardware) <br> (B) Use SQLite (not realistic) | **Choice:** _______ | DevOps |
| 6.2 | **On-Call Escalation Matrix** | (A) P1: Backend Lead, P2: DevOps, P3: FE Lead <br> (B) Rotating schedule (all devs) | **Choice:** _______ <br> **Primary On-Call:** _______ | CTO |
| 6.3 | **Feature Flag Strategy** | (A) Use Django Admin `FeatureFlag` table <br> (B) Hardcode `if DEBUG` (messy) | **Choice:** _______ | Backend Lead |
| 6.4 | **Error Reporting to Teachers** | (A) Bilingual (English + Telugu) UI messages <br> (B) Only English | **Choice:** _______ | Product / FE Lead |

---

## CHAIR'S SUMMARY & FINAL SIGN-OFF

**Recap of Biggest Risks Mitigated Today:**
1. **Financial:** We locked the pricing tier and set cost-alert thresholds to prevent cloud bill shocks.
2. **Performance:** We committed to monthly DB partitioning and aggressive CDN caching to handle 1M schools.
3. **Reliability:** We mandated monthly "Chaos Drills" to test RTO/RPO, ensuring we don't panic during a real outage.
4. **Team Safety:** The CI/CD OpenAPI hard gate prevents backend/frontend integration surprises.

**Action Items for the Chair to Distribute Post-Meeting:**
- Send this signed PDF to all leads.
- Create a `docs/decisions.md` in the GitHub repository with these decisions (Architecture Decision Records - ADRs).
- Schedule a follow-up in 2 weeks to check progress on the critical path items (Subdomain Middleware, Partitioning Script, and CI/CD Pipeline).

**Chair Signature:** _________________________  
**Date:** _________________________

---

*This document is now the constitution of the project. Any deviation from these signed decisions requires a formal Change Request approved by the CTO and CEO.*
