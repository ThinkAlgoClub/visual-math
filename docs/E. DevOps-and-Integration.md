## E. CROSS-CUTTING / DEVOPS & INTEGRATION – PRACTICAL CI/CD, COST & DNS

---

### E1. CI/CD Pipeline Architecture (The "Build, Test, Deploy" Assembly Line)
*Decision needed: A broken `openapi.json` or a failed migration must stop the deployment train immediately.*

- **Branch Strategy & Environment Mapping:**
  - **Decision:** Adopt **GitHub Flow** (or GitLab Flow) with three permanent environments:
    - `main` → Deploys to **Staging** (automatic).
    - `production` → Deploys to **Production** (manual approval required via GitHub Environments).
    - Feature branches → Trigger only `lint` + `test` (no deploy).
- **Pipeline Stages (Exact Order & Failure Conditions):**
  - **Stage 1: Lint & Security Scan (2 mins):**
    - Backend: `flake8`, `black --check`, `bandit` (security scanner for Python).
    - Frontend: `eslint`, `prettier --check`.
    - *Fail Condition:* Any warning → Pipeline fails. No exceptions.
  - **Stage 2: Unit & Integration Tests (5 mins):**
    - Backend: `pytest` with `--cov` (minimum 85% coverage enforced).
    - Frontend: `jest` (or `vitest`) for vanilla JS modules.
    - *Critical:* Tests must run against a **temporary PostgreSQL container** (GitHub Actions `postgres` service) with the latest migrations applied. This catches migration errors early.
  - **Stage 3: OpenAPI Contract Enforcement (The FE/BE Handshake):**
    - Backend generates `openapi.json` using `drf-spectacular` and uploads it as a pipeline artifact.
    - Frontend runs a script (`openapi-typescript` or a custom `validate-contract.js`) that compares the frontend's API call definitions (stored in `src/api/endpoints.js`) against the new schema. If any endpoint URL, method, or required field is missing/mismatched, the pipeline **hard fails**.
    - *Why this matters:* Prevents the classic "works on my machine" integration disaster.
  - **Stage 4: Build & Bundle (3 mins):**
    - Backend: Collects static files, compresses them (`gzip`/`brotli`).
    - Frontend: Bundles the main `app.js` and creates versioned hashes (e.g., `app.a1b2c3.js`) for long-term caching.
  - **Stage 5: Deploy (Manual for Prod):**
    - Pushes Docker images to AWS ECR (Elastic Container Registry).
    - Triggers AWS ECS (Fargate) or Kubernetes rolling update.

- **CI Runner Specs (Cost vs Speed):**
  - **Decision:** Use GitHub Larger Runners (4-core, 16GB RAM) for the Build stage (Three.js bundling is CPU-heavy). Use standard 2-core runners for test stages to save costs.

---

### E2. Zero-Downtime Deployment Strategy (The "No Teacher Notices" Rule)
*Decision needed: We cannot restart all Gunicorn workers at once—teachers will see 502 errors.*

- **Rolling Update Strategy (AWS ECS / Kubernetes):**
  - **Decision:** Use **AWS ECS Rolling Update** with `maximumPercent = 200` and `minimumHealthyPercent = 100`. This spins up new containers *before* killing the old ones.
  - **Health Check Grace Period:** Set the container health check to hit `/api/v1/health/` (which checks DB and Redis connectivity). Give it 60 seconds of grace period before ECS decides it's unhealthy.
- **Database Migration Order (The Chicken-Egg Problem):**
  - **Golden Rule:** *Always* deploy code that is *backward-compatible* with the *old* DB schema.
  - **Migration Run Schedule:** Run `python manage.py migrate` **BEFORE** the new application containers start serving traffic.
  - *Practical Flow:*
    1. Pipeline runs `migrate` against the Prod DB (which is safe if we followed the "3-step migration" rule from Section D).
    2. After migration succeeds, trigger the ECS rolling update.
    3. Old containers (running old code) can still read/write to the DB because the new columns are nullable/optional.
    4. New containers roll in; old containers roll out.
- **Rollback Plan (The "5-Minute Rule"):**
  - **Decision:** If we detect a critical bug post-deploy, the rollback is a **single click** in ECS: "Revert to previous task definition".
  - *Caveat:* If the bug involves a DB schema change that **deletes** data, rollback is impossible without restoring from backup. Therefore, we enforce the "Add → Backfill → Drop" cycle strictly to avoid destructive schema changes.

---

### E3. DNS, Subdomain Routing & Wildcard SSL (The "Never Expire" Setup)
*Decision needed: We agreed to use a Wildcard SSL to avoid Let's Encrypt rate limits, but how do we actually route 1M subdomains?*

- **DNS Configuration (The A Record):**
  - **Decision:** Add a single `A` (or `CNAME`) record for `*.mathvision.com` pointing to the AWS Load Balancer (ALB) with a **TTL of 60 seconds**.
  - *Why 60s?* If we ever need to change the Load Balancer IP (disaster recovery), we want DNS propagation to happen in minutes, not hours.
- **Reverse Proxy Configuration (Caddy vs Nginx):**
  - **Decision:** Use **Caddy** as the reverse proxy sidecar in the ECS task.
  - **Why Caddy?** It handles wildcard SSL termination automatically (using ACME + Let's Encrypt) without complex Lua scripts. Caddy reads the `Host` header, extracts the subdomain, and forwards it to the Django app running on `localhost:8000`.
  - **Exact Caddyfile:**
    ```caddy
    *.mathvision.com {
        tls {
            dns route53 {env.AWS_ACCESS_KEY_ID} {env.AWS_SECRET_ACCESS_KEY}
        }
        reverse_proxy localhost:8000 {
            # Forward the original host header to Django for subdomain middleware
            header_up Host {host}
        }
    }
    ```
  - *Failure Mode:* If Caddy fails to renew the wildcard cert (rare), we must have a fallback `http`→`https` redirect. Caddy retries renewal automatically; we just monitor logs.
- **Subdomain Routing inside Django:**
  - The `SubdomainMiddleware` (Section B2) will parse `request.get_host()` to extract the subdomain and attach the `school` object. Since Caddy passes the original Host header, Django sees the correct subdomain.

---

### E4. Environment Parity & Secrets Management (The "No Hardcoding" Rule)
*Decision needed: A developer committing an AWS key to Git is an instant security breach.*

- **Environment Variables (12-Factor App):**
  - **Decision:** Use **AWS Secrets Manager** for all production secrets (DB passwords, RazorPay keys, Django `SECRET_KEY`, Redis passwords).
  - **Dev/Staging:** Use `.env` files stored in the repository (but `.gitignored` from actual values). The CI pipeline injects them via GitHub Secrets.
- **Container Environment Variable Injection:**
  - ECS Task Definitions reference the Secrets Manager ARNs. Example: `Django_Secret_Key` is pulled from `arn:aws:secretsmanager:...`.
- **Staging vs Production Isolation:**
  - Staging uses a separate AWS RDS instance (smaller, cheaper), separate S3 bucket, and **RazorPay Test Keys**.
  - **Hard Rule:** The Staging environment *must* mirror Production OS versions (Ubuntu 22.04, Python 3.11) even if the hardware is smaller.

---

### E5. Cost Monitoring & Budget Alerts (The "Awake at 2 AM" Alarms)
*Decision needed: A DDoS attack or misconfigured CDN could cost ₹10 Lakhs in a day. We need financial circuit breakers.*

- **AWS Cost Allocation Tags:**
  - Tag every resource (EC2, RDS, S3, CloudFront) with `Project: MathVision` and `Environment: Prod`.
  - Create an **AWS Budget** with alerts at **80%** and **100%** of the monthly projected cost.
- **CloudFront Egress Cost Mitigation (The Silent Killer):**
  - Heavy Three.js libraries (600KB) delivered 100,000 times/day = 60GB * $0.09 = ~$5.4/day = ~$162/month. Acceptable, but we need to minimize it.
  - **Decision:** Set **Cache-Control headers** aggressively:
    - `max-age=31536000, immutable` for JS/CSS bundles (only changes when the file hash changes).
    - `max-age=3600, must-revalidate` for JSON lesson configs.
  - **Cost-Cutting Measure:** Enable **CloudFront Origin Shield** to consolidate cache hits across edge locations, reducing the number of requests hitting the Django backend (which also saves compute costs).
- **RDS Storage Cost Alerts:**
  - If `TeacherActivity` partitions grow faster than expected, storage costs will spike.
  - **Decision:** Set a CloudWatch Alarm if RDS `FreeStorageSpace` drops below 20% of provisioned capacity. This triggers an automation script (Lambda) to increase storage automatically (AWS supports auto-scaling storage for RDS).
- **The "Kill Switch" (Last Resort):**
  - If monthly costs exceed 150% of budget, an automated Lambda function **could** throttle the application to "Read-Only" mode for **unpaid/expired schools only**, preventing high-traffic non-revenue schools from burning cash. *This requires stakeholder approval.*

---

### E6. Observability, Logging & Actionable Alerting (The "Who to Page" Matrix)
*Decision needed: Nagging alerts cause alert fatigue. We want alerts that wake the right person only when the teacher is actually suffering.*

- **Log Aggregation Strategy:**
  - **Decision:** Stream all container logs to **AWS CloudWatch Logs**.
  - For structured JSON logs (from Section B6), use CloudWatch Logs Insights to query for `status:500` or `error_code:E_SUB_EXPIRED`.
  - **Retention:** Store logs for 30 days in CloudWatch, then archive to **S3 Glacier** for compliance (1-year minimum).
- **Application Performance Monitoring (APM):**
  - **Decision:** Use **DataDog** or **NewRelic** (choose based on budget; DataDog is expensive but excellent for Python/Django).
  - **Instrumentation:** Install the `ddtrace` library. Automatically instrument all Django views, DB queries, and Redis/Celery tasks.
  - **SLO Dashboard:** Create a dashboard tracking:
    - API P95 latency (target: < 300ms for subdomain check, < 800ms for simulation data).
    - Error Rate (target: < 0.5%).
    - Celery queue backlog.
- **On-Call Alert Rules (PagerDuty / Opsgenie):**
  | Alert Condition | Severity | Action |
  | :--- | :--- | :--- |
  | `error_rate > 2%` for 2 consecutive minutes | P1 (Critical) | Page Backend Lead (SMS + Call) |
  | `RDS CPU > 80%` for 5 minutes | P2 | Page DB Lead (Slack only) |
  | `Redis memory usage > 85%` | P2 | Page DevOps (Slack) |
  | `Celery queue backlog > 1,000 tasks` | P3 | Page Backend (Email) |
  | `CloudFront 5xx rate > 5%` | P1 | Page DevOps (Call) |
- **Sentry (Frontend Error Tracking):**
  - **Decision:** Integrate Sentry for Browser JS errors.
  - *Alert:* If a new issue (e.g., "TypeError: three is undefined") occurs > 5 times in 1 minute, send a P2 alert to the Frontend Lead.
  - *Bonus:* Sentry automatically captures the `correlation_id` from the response header and links it to the backend logs.

---

### E7. Infrastructure as Code (IaC) & Disaster Recovery Drills (The "Burn the Staging" Test)
*Decision needed: If a region goes down, how fast can we move?*

- **IaC Tool:** **Terraform** (or AWS CDK).
  - **Decision:** All infrastructure (VPC, subnets, RDS, ECS cluster, S3 buckets, CloudFront) is defined in Terraform code.
  - *Why?* We can spin up a full "Disaster Recovery" (DR) region in another AWS AZ in under 30 minutes.
- **The "Chaos Monday" Drill (Monthly):**
  1. **Terminate 50%** of the running ECS containers randomly (simulate instance failure).
  2. Observe if the Auto-Scaling group recovers within 2 minutes.
  3. **Simulate RDS Primary Failure:** Force a failover to the read-replica. Measure downtime (should be < 60 seconds).
  4. **Restore from PITR:** In the Staging environment, run the full restore drill (Section D6) and time it. If it exceeds 30 minutes, the process is considered "failed" and must be re-optimized.
- **Region Failover Plan:**
  - If the entire `ap-south-1` (Mumbai) region goes down:
    - We have a hot-standby environment in `ap-southeast-1` (Singapore).
    - Update the DNS `A` record TTL to point to the Singapore Load Balancer.
    - **RTO:** 15 minutes (manual promotion). **RPO:** 5 minutes (async WAL replication across regions—though high latency, we accept this).

---

### E8. Cache Invalidation & Static Asset Versioning (The "Update the Lesson" Headache)
*Decision needed: Content admins upload a new simulation config. Teachers still see the old one because CloudFront cached it.*

- **Asset Versioning Strategy (The "Cache Buster"):**
  - **Decision:** Append a query string to all static asset URLs based on the `GIT_COMMIT_SHA`.
    - Example: `<link rel="stylesheet" href="{% static 'css/main.css' %}?v=abc1234">`
    - Django's `staticfiles` app with `ManifestStaticFilesStorage` automatically does this.
  - **For CDN (CloudFront):** When the backend deploys, invalidate the CloudFront cache for the specific files (`/js/main.*.js`, `/css/*.css`) using the AWS CLI: `aws cloudfront create-invalidation --paths "/js/*" "/css/*"`.
- **Simulation Config Caching (The tricky one):**
  - Teachers need the *latest* simulation config, but we also want to cache it for performance.
  - **Decision:** Use `Cache-Control: max-age=60, must-revalidate` for the API endpoint `/api/v1/simulation/config/{topic_id}/`.
  - When the Admin updates the config via Django Admin, we trigger a `post_save` signal that instantly purges the Redis cache for that topic (`cache.delete(f'topic_config:{topic_id}')`). The next request will hit the DB and repopulate Redis.

---

### DevOps / Cross-Cutting Sign-Off Checklist (To Leave the Meeting With)

| # | Action Item | Owner | Deadline |
| :--- | :--- | :--- | :--- |
| 1 | Set up GitHub Actions (or GitLab CI) with the 5-stage pipeline (Lint → Test → Contract → Build → Deploy). Add `bandit` and `eslint` checks. | DevOps Lead | Sprint 1 |
| 2 | Implement the **OpenAPI Contract Validation** step in the pipeline (Backend generates `openapi.json`, Frontend validates against it). | Backend + FE Lead | Sprint 2 |
| 3 | Configure **Caddy** as the reverse proxy with Wildcard SSL (`*.mathvision.com`) in the ECS task definition. | DevOps Lead | Sprint 1 |
| 4 | Set up AWS Secrets Manager for all production keys and inject them into ECS containers. | DevOps Lead | Sprint 1 |
| 5 | Establish **AWS Cost Allocation Tags** and create Budget Alerts at 80% / 100% of projected monthly spend. | DevOps Lead | Immediate |
| 6 | Integrate **DataDog/NewRelic APM** and configure the On-Call alert rules in PagerDuty (Slack + SMS). | DevOps Lead | Sprint 2 |
| 7 | Set up **Sentry** for Frontend JS error tracking and link it to Backend `correlation_id`. | FE + Backend Lead | Sprint 2 |
| 8 | Schedule the first **"Chaos Monday"** Disaster Drill (simulate RDS failover and ECS termination) in Staging. Document the runbook. | DevOps Lead | End of Sprint 3 |
| 9 | Configure **CloudFront Invalidations** to run automatically on production deployments. | DevOps Lead | Sprint 1 |
| 10 | Define the **Rollback Playbook**: Document the exact steps (ECS Revert, DB rollback limits, and the "break glass" procedure for emergency). | DevOps + CTO | Immediate |
