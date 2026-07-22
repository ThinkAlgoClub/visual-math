## A. BUSINESS & STAKEHOLDER – PRACTICAL PROCUREMENT, POLICY & ROLLOUT (DEEP DIVE)

---

### A1. Procurement & Billing Mechanics (How money actually moves)
*Decision needed: We are building for Indian government and private school bureaucracy—not e-commerce.*

- **Invoice Generation & GST Compliance:** 
  - Schools in India require a formal tax invoice with GSTIN (if registered) before releasing payment. 
  - **Practical Decision:** Does RazorPay handle invoice generation, or does our Django app generate a PDF invoice via `weasyprint`/`reportlab` *before* redirecting to RazorPay? If we generate it, we must store the invoice number sequence (e.g., `INV/AP/2026/0001`) in the DB and handle GST split (CGST+SGST 9% each or IGST 18% for inter-state).
  - **Escalation:** What if a school asks for a **Physical Purchase Order (PO)** with a 30-day credit period? RazorPay doesn't support credit. *Decision:* Do we build a separate "Offline Payment" workflow where the Principal uploads a scanned PO, and our finance team manually marks the subscription as "active" after receiving the cheque/UPI? (This requires a Django Admin "Manual Activation" button with audit logs).
- **Trial Period & Auto-Expiry Logic:**
  - We agreed on a 7-day free trial. But what counts as "Day 1"? 
  - **Practical Decision:** The trial should start the moment the Principal *creates the subdomain and verifies their email*, NOT when they add teachers. If they don't pay by Day 7, the subdomain goes into `status="EXPIRED"`. However, we must send **Dunning emails** (Day 3, Day 5, Day 6) via Celery to nudge them.
  - **Grace Period Mechanics:** After expiry, we give 7 days of read-only access. *Read-only means:* They can see the dashboard and lesson lists but clicking on any simulation shows a blurred overlay with a "Reactivate Now" button. The backend API must return `403 FORBIDDEN` with error code `E_SUB_EXPIRED` for any simulation rendering endpoint.
- **Renewal & Failed Payment Handling:**
  - RazorPay offers "Subscriptions" with auto-debit mandates. 
  - **Decision:** Do we force auto-renewal, or send a manual payment link 15 days before expiry? *Recommendation:* Use manual renewal for Year 1 to avoid failed mandate disputes. On expiry, the Principal receives an email with a unique RazorPay payment link (generated via their API). If they don't renew within 7 days, we soft-delete the school's subdomain (release it back to the pool after 30 days).

---

### A2. State Policy & Curriculum Compliance (The Non-Technical Risk)
*Decision needed: Government schools face audits. We must prove our content is 100% aligned.*

- **SCERT / Board Approval Status:** 
  - Telangana and AP have their own State Councils of Educational Research and Training. 
  - **Practical Action:** Do we have a signed MOU/approval from both state education departments? If not, teachers may be reluctant to use it. We need a **"Content Verification Log"** in the Admin panel—a date-stamped PDF uploaded for each chapter, signed by our internal subject matter expert, proving the simulation aligns with the latest syllabus.
  - **Rollout Decision:** If AP approves but Telangana is pending, do we launch AP-only initially? This affects database partitioning—we must design the `School.state` field to allow only approved states, with a feature flag to toggle visibility.
- **Medium of Instruction (Bilingual UI):**
  - The UI buttons and error messages must be in **English and Telugu** (for both states). 
  - **Practical Decision:** We are not building a full i18n library. Instead, store all static UI text in a `translations.json` file on the CDN, keyed by `state` (AP uses Telugu script, TG uses Telugu + Urdu?). The Principal selects the language preference during onboarding, and the frontend loads the appropriate JSON.
- **Textbook Edition Year Conflicts:**
  - AP and TG update textbooks every 3-4 years. A school might still follow the 2023 edition while another follows 2026. 
  - **Decision:** The `Curriculum` table must have an `academic_year` field (e.g., `2025-26`). The Principal must select the academic year during onboarding. We must support multiple editions running in parallel. If a teacher logs in and their school uses the old edition, they see the old simulations. This is a massive content maintenance effort—how often will we backport new simulations to old editions? (Likely never—so we need a sunset policy: old editions expire after 2 years).

---

### A3. Rollout Phasing & District-Level Deployment (The Ground Reality)
*Decision needed: We cannot turn on 1M schools on Day 1. We need a controlled blast radius.*

- **Pilot Phase (Month 1-3):**
  - Target **50 schools** (25 in Hyderabad, 25 in Visakhapatnam) with a dedicated relationship manager. 
  - **Practical Decision:** The registration flow must have a "Pilot Code" field (optional). Pilot schools get a 6-month free subscription, and we track their usage via a `is_pilot` boolean in the `School` model. This allows us to throttle support tickets and fix bugs before the general public launch.
  - **Success Criteria:** We must define "Pilot Success" as >80% of teachers using the app at least 3 times a week. If we don't hit this, we pause the public launch and rework UX.
- **District-Wise Progressive Rollout:**
  - Instead of opening to the entire state, we roll out district-by-district (e.g., first Rangareddy, then Guntur). 
  - **Decision:** We implement a `restricted_districts` array in Django Admin. If a Principal registers from a non-approved district (checked via their registered school pin-code), the signup shows "Coming soon to your district" and collects their email for a waitlist. This prevents a support avalanche.
- **Training & Change Management:**
  - Teachers in rural areas will need a 1-hour physical/virtual training session. 
  - **Decision:** Who conducts this? We need an integration with a third-party training partner. The app must have a **"Training Mode"** where an on-screen coach (video + highlighted UI pointers) walks the teacher through the first simulation. This mode is triggered automatically on the teacher's first login.
- **Hardware Readiness Survey:**
  - Most schools have a single smartboard/projector connected to a PC. 
  - **Decision:** We must add a mandatory survey during Principal onboarding: "What is your classroom display resolution?" (e.g., 1024x768, 1920x1080, 4K). The frontend will adapt its DPI scaling accordingly. If 80% of users are on 1366x768 laptops, we design the UI for that, not for 4K monitors.

---

### A4. Pricing Strategy, Competition & Revenue Forecasting
*Decision needed: How do we justify the price to a budget-constrained school?*

- **Competitive Landscape:**
  - Byju's, Toppr, and other ed-tech platforms are either B2C or expensive. We are B2B (schools).
  - **Decision:** What is our annual price point? (e.g., ₹5,000/year, ₹10,000/year, or ₹25,000/year?). This determines whether we target *Private Schools* (willing to pay) vs *Government Schools* (need government subsidy). 
  - *Practical advice:* If we target Government Schools, we must integrate with the **PFMS (Public Financial Management System)** portal for direct payments, which is a 6-month integration project. We need a separate offline invoicing module for government tenders.
- **Revenue Recognition (Accounting):**
  - Since we collect annual payments upfront, we cannot book it as revenue on Day 1 (GAAP/IFRS rules). 
  - **Decision:** We must track `deferred_revenue` in the DB. When the payment succeeds, we create a `Subscription` record with `amount_received`, `start_date`, `end_date`. A monthly cron job runs to amortize revenue: `recognized_revenue += total_amount/12`. This is crucial for investor/financial reporting.
- **Churn Rate & Retention Metrics:**
  - What is the acceptable churn rate? (e.g., <15% annually).
  - **Decision:** We must build a `ChurnPrediction` module that flags schools that haven't logged in for 30 days. Our sales team gets an automatic notification to call the Principal and offer a discount for renewal. This is a simple Django Admin list filter: `last_login < now() - 30 days`.

---

### A5. Data Privacy, DPDP Act & Parental Consent (Legal Landmines)
*Decision needed: We store teacher data, but what about accidental student data entry?*

- **Teacher Data Retention Policy:**
  - We store teacher names, email, and phone numbers (if provided). Under India's DPDP Act, we are a "Data Fiduciary".
  - **Decision:** We must add a "Data Retention Period" setting (e.g., 3 years after subscription ends). A Celery daily task must identify schools that expired > 3 years ago and **anonymize** the teachers' personal data (replace name/email with `deleted_user_{uuid}`) while keeping aggregated usage statistics for analytics.
- **Accidental Student Data Exposure:**
  - Teachers might write a student's name on the digital canvas (annotation) and save it.
  - **Decision:** The Canvas annotation data is stored as JSON vectors (not images). We cannot automatically scrub names. Therefore, in the Privacy Policy, we must explicitly state that the school is responsible for not inputting PII (Personally Identifiable Information) on the canvas. We need a check-box agreement during signup.
- **Subdomain Takeover & Brand Impersonation:**
  - If a malicious actor registers `delhi-public-school.mathvision.com` and creates a fake brand, they could scam teachers.
  - **Decision:** We need a **"Verify School"** badge. A Principal can request verification by uploading their school registration certificate (PDF). Once a super-admin approves it, a green tick appears next to the school name in the dashboard. This builds trust.

---

### A6. Escalation Matrix & Crisis Communication (When Things Break)
*Decision needed: What happens when the app crashes during a live class?*

- **Incident Response SLA:**
  - We cannot have a 24-hour fix time during school hours (8 AM – 4 PM).
  - **Decision:** Define severity levels:
    - **P0 (Total outage):** Fix within 1 hour. On-call engineer rotation must be established.
    - **P1 (Subdomain SSL error):** Fix within 4 hours.
    - **P2 (UI bug, e.g., slider not moving):** Fix within 48 hours.
  - We must build a **Status Page** (e.g., `status.mathvision.com`) that teachers can check if the app is down, instead of flooding the support email.
- **Complaint Redressal Mechanism:**
  - Schools may complain about incorrect content (e.g., "This graph shows 45 degrees but the textbook says 30").
  - **Decision:** Build a "Report Issue" button inside every simulation. This opens a modal that captures the simulation state, the current URL, and allows the teacher to draw a red circle around the error. This report goes directly to the Content Team's Slack/Discord channel via a webhook, allowing a 24-hour turnaround on fixes.

---

### Final Stakeholder Sign-Off Checklist (To Leave the Meeting With)

| # | Action Item | Owner | Deadline |
| :--- | :--- | :--- | :--- |
| 1 | Decide Annual Price (₹5k / ₹10k / ₹25k) + define Basic vs Premium tier limits. | CEO / Sales Head | End of Week |
| 2 | Confirm Pilot District list and acquire official MOU from AP/TG SCERT. | Legal / Govt. Relations | 15 Days |
| 3 | Finalize the "Offline Payment" workflow (PO + manual activation) – Yes/No? | Finance / CTO | 3 Days |
| 4 | Approve DPDP consent text and Data Retention Policy (3-year anonymization). | Legal Team | 1 Week |
| 5 | Set the Pilot Success Criteria (e.g., 80% weekly active teachers) before public launch. | Product Manager | Immediate |
| 6 | Determine the exact Trial Period (7 days) and Grace Period (7 days read-only). | CEO / Sales | Immediate |
| 7 | Sign off on the "Training Mode" development effort (adds 2 sprint points). | CTO / Product | End of Sprint Planning |
