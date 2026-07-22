## B. BACKEND TEAM – PRACTICAL CODE, MIDDLEWARE & API FAILURE MODES

---

### B1. Authentication & Session Management (Stateless JWT with Hard Invalidation)
*Decision needed: We must support instant revocation without hitting the database on every request.*

- **Token Implementation Choice:**
  - **Decision:** Use `djangorestframework-simplejwt`. *Do NOT* use Django's built-in `SessionAuthentication` for APIs (stateful, hard to scale horizontally).
  - **Token Structure:** Access Token TTL = **6 hours** (covers a full school day). Refresh Token TTL = **7 days** (allows teachers to stay logged in over the weekend without re-entering password).
- **Hard Invalidation (The `last_password_change` Trick):**
  - Add a `last_login_or_pw_change` DateTimeField to the `User` model. Update this field on every successful login AND when the Principal resets a teacher's password.
  - **Code Implementation:** Create a custom DRF Authentication class that overrides `authenticate()`:
    ```python
    # Simplified logic
    if user.last_login_or_pw_change > datetime.fromtimestamp(payload.get('iat')):
        raise AuthenticationFailed('Token invalid. Password or login state changed.')
    ```
  - **Failure Mode:** If the DB fails to update `last_login_or_pw_change` due to a deadlock, the token remains valid longer than intended. *Mitigation:* Wrap the update in a `select_for_update()` lock or use a dedicated Redis cache for blacklisted tokens as a secondary check.
- **Logout (Server-Side vs Client-Side):**
  - Since JWTs are stateless, "Logout" is purely client-side (deleting the cookie). However, we cannot truly invalidate the token server-side unless we maintain a blacklist. 
  - **Decision:** We will implement a **soft-logout**. When a teacher clicks "Logout", we call an API endpoint `/api/v1/auth/logout/` that simply adds the JWT's `jti` (JWT ID) to a Redis blacklist with TTL = 6 hours (the remaining token lifetime). This prevents token reuse even if the cookie is stolen. The blacklist check is a 2ms Redis GET—negligible overhead.

---

### B2. Multi-tenancy Middleware (Subdomain Routing & Caching)
*Decision needed: We cannot query the DB for `School` on every static file request.*

- **Middleware Execution Order:**
  - Place `SubdomainMiddleware` as the **first middleware** in `MIDDLEWARE` list (above SecurityMiddleware). Reason: We need the `school` object injected into the request before authentication or any view logic.
- **Redis Cache-Aside Pattern (Exact Code Flow):**
  ```python
  def __call__(self, request):
      host = request.get_host().split(':')[0] # Remove port
      subdomain = host.split('.')[0] if host.endswith('mathvision.com') else None
      
      if subdomain and subdomain not in ['www', 'app', 'api', 'admin']:
          # Step 1: Check Redis
          school_id = cache.get(f'subdomain:{subdomain}')
          if not school_id:
              # Step 2: DB Fallback
              try:
                  school = School.objects.only('id', 'state', 'subscription_status').get(subdomain=subdomain)
                  cache.set(f'subdomain:{subdomain}', school.id, timeout=86400) # 24 hours
              except School.DoesNotExist:
                  # Return a 404 page WITHOUT redirecting (prevents subdomain takeover attacks)
                  return HttpResponseNotFound("School not found. Please check the URL.")
          
          # Step 3: Attach school to request
          request.school = school 
      return self.get_response(request)
  ```
- **Cache Invalidation (When Principal changes subdomain? We never allow changes, so no invalidation needed here).** 
  - However, if the school's `subscription_status` changes (payment), we must invalidate the cache. **Decision:** Use Django's `post_save` signal on the `School` model to call `cache.delete(f'subdomain:{subdomain}')`.
- **Failure Mode (Redis Down):** 
  - The middleware will fallback to DB queries. This will hammer the DB. 
  - **Mitigation:** Wrap the cache/DB logic in a `try-except` and log a critical alert to Sentry. If Redis is down for > 1 minute, we trigger an automatic failover to a read-replica DB for these lookups.

---

### B3. API Design, Versioning, and Centralized Error Handling (The Single Source of Truth)
*Decision needed: Frontend must show human-readable messages in English/Telugu without hardcoding them.*

- **API Versioning Strategy:** 
  - **Decision:** Use **URL versioning** (`/api/v1/...`, `/api/v2/...`). This is the most explicit and cache-friendly. 
  - *Policy:* We will maintain backward compatibility for 2 major versions (e.g., support v1 and v2 for 12 months after v3 releases). 
- **Standardized Error Response Structure (Contract):**
  All 4xx/5xx responses must follow this exact JSON schema:
  ```json
  {
    "error_code": "E_SUB_EXPIRED",
    "message": "Your school subscription has expired. Please renew to access simulations.",
    "details": { "expiry_date": "2026-07-15" },
    "trace_id": "a1b2c3d4-e5f6-7890" // Correlation ID
  }
  ```
- **Centralized Exception Handler (DRF Custom Handler):**
  ```python
  # Override EXCEPTION_HANDLER in settings
  def custom_exception_handler(exc, context):
      # Handle Django's ValidationError, PermissionDenied, Http404
      # Map them to custom error codes.
      if isinstance(exc, SubscriptionExpired):
          return Response({"error_code": "E_SUB_EXPIRED", ...}, status=403)
      if isinstance(exc, Throttled):
          return Response({"error_code": "E_RATE_LIMIT", "details": {"wait": exc.wait}}, status=429)
      # Unhandled exceptions -> 500 with trace_id only (no stack trace exposed to FE)
      return Response({"error_code": "E_INTERNAL", "trace_id": get_current_trace_id()}, status=500)
  ```
- **Shared Error Codes:** Store `error_codes.json` on the S3/CDN. The Frontend loads this to map `E_SUB_EXPIRED` to a Telugu/English sentence. The Backend only returns the `error_code`. This decouples UI language updates from Backend deployments.

---

### B4. Celery Task Orchestration (Idempotency, Retries, and Dead Letter Queue)
*Decision needed: What happens when an async task fails halfway? We need mathematical precision on retries.*

- **Task Breakdown:**
  1. `send_welcome_email`: TTL 10 seconds.
  2. `generate_subdomain_certificate`: (Deprecated, we use Wildcard now, but replaced by `generate_school_report_pdf` or `sync_curriculum_assets`).
  3. `process_razorpay_webhook`: Critical.
- **RazorPay Webhook Idempotency (The 3-Layer Shield):**
  - **Layer 1 (DB Unique Constraint):** Add `razorpay_payment_id` as a `UniqueConstraint` on the `Payment` model. If the same webhook arrives twice, `integrity_error` is raised and caught.
  - **Layer 2 (Task Deduplication):** Use `celery_once` library or Redis lock with the `payment_id` as the lock key (TTL = 1 hour) to prevent parallel execution of the same webhook.
  - **Layer 3 (State Machine):** The `Subscription` model has a `status` field (`PENDING`, `ACTIVE`, `PROCESSING`, `FAILED`). The task checks: `if subscription.status == 'ACTIVE': return` (idempotent exit).
- **Exponential Backoff Policy (The Golden Rule):**
  ```python
  @app.task(bind=True, max_retries=5, soft_time_limit=60, hard_time_limit=120)
  def activate_subscription(self, payment_id):
      try:
          # 1. Activate DB, 2. Send email, 3. Push to analytics
          pass
      except RazorpayAPITimeout as e:
          # Retry after 2^retry_count * 60 seconds (60s, 120s, 240s, 480s, 960s)
          raise self.retry(exc=e, countdown=60 * (2 ** self.request.retries))
  ```
- **Dead Letter Queue (DLQ) Strategy:**
  - After 5 failed retries, the task goes to the DLQ.
  - **Decision:** We need a Celery worker dedicated to DLQ tasks that simply **sends a critical Slack/Email alert to the Ops team** (e.g., "Webhook processing failed for payment X. Manual intervention required"). No automated retry after 5 failures—human intervention is mandatory.

---

### B5. Database Query Optimization (N+1 Nightmare & Selective Loading)
*Decision needed: Prevent the ORM from killing the DB under load.*

- **Django ORM Hard Rule:** *No* `.objects.all()` or `.objects.get()` without explicit `.only()` or `.defer()` in simulation rendering endpoints.
  - **Example:** When fetching the lesson tree, we only need `id`, `name`, `slug`, and `parent_id`. We do NOT need the 500KB `simulation_config_json` field.
  - **Decision:** Use `ModelSerializer` with `fields = ['id', 'name', 'slug']` and explicitly query `Curriculum.objects.filter(...).only('id', 'name', 'slug', 'parent')`.
- **N+1 Query Detection (CI Enforcement):**
  - Integrate **django-debug-toolbar** in staging.
  - Set a rule: Any API endpoint that executes > 10 queries per request (excluding cache hits) will cause the CI/CD pipeline to **fail the build**. We will enforce this using `nplusone` library or `assertNumQueries` in unit tests.
- **Database Indexing (Enforced via Migration):**
  - Add `db_index=True` to `School.subdomain`, `Curriculum.state`, `Curriculum.class_level`, `User.school_id`.
  - **Decision:** For the `TeacherActivity` table (partitioned), we will create a composite index on `(school_id, topic_id, created_at DESC)` to speed up "last viewed" queries.

---

### B6. Request/Response Logging & Correlation ID (Tracing a User's Journey)
*Decision needed: When a teacher reports a bug, we need to find their specific request instantly.*

- **Correlation ID Generation:**
  - Create a middleware that injects a `X-Request-ID` header. If the client sends one, we use it; otherwise, we generate a UUID.
  - **Code:** `request.correlation_id = request.META.get('HTTP_X_REQUEST_ID', str(uuid4()))`
  - Add this to every log line using Django's `logging` `extra` filter.
- **Structured Logging (JSON Logs):**
  - **Decision:** Switch Django logging to output **JSON** (using `python-json-logger`) instead of plain text. 
  - *Why?* When we deploy to AWS CloudWatch / Datadog, JSON logs are automatically parsed into searchable fields.
  - *Log Level:* 
    - `INFO`: Every API request (method, path, status_code, duration_ms).
    - `WARNING`: Throttling hits, failed token validation.
    - `ERROR`: 500 errors (include full stack trace only in ERROR logs, not in response).
    - `CRITICAL`: DB connection failures, Redis outages.

---

### B7. Rate Limiting & Throttling (Protecting the Math Engine)
*Decision needed: Concrete numbers to prevent a single school from causing a cascade failure.*

- **Django Ratelimit Setup:** Use `django-ratelimit` or DRF's built-in throttling.
- **Tier 1: Unauthenticated (Subdomain Check):**
  - Endpoint: `/api/v1/subdomain/check/`
  - **Limit:** 10 requests per minute per IP. 
  - *Reason:* Prevent bots from enumerating subdomains.
- **Tier 2: Simulation Assets (The heavy lifters):**
  - Endpoint: `/api/v1/simulation/data/{topic_id}/` 
  - **Limit:** 100 requests per minute per **School** (not per user). Since 5 teachers share one school, this allows each teacher to open ~2 new lessons per minute, which is reasonable.
  - *Implementation:* Use the `request.school.id` as the throttle key.
- **Failure Mode (Throttled):** Return `429 Too Many Requests` with a `Retry-After` header (seconds). The Frontend must listen for this and show a countdown timer to the teacher, preventing blind retries that worsen the load.

---

### B8. Security Hardening (Environment Variables, Command Injection, Raw SQL Ban)
*Decision needed: Explicitly forbid dangerous practices.*

- **Environment Variables (The 12-Factor App Rule):**
  - No hardcoded secrets in `settings.py`. Use `django-environ` to read from `.env`.
  - **Decision:** Separate `.env` files per environment (`dev.env`, `staging.env`, `prod.env`) stored in AWS Secrets Manager, not in the code repository.
- **Ban Raw SQL & Shell Commands:**
  - **Hard Rule:** Any use of `RawSQL`, `connection.execute()`, or `subprocess.Popen` in the codebase must pass a **security review** by the Lead Architect and be heavily commented with justification. Otherwise, we strictly use Django ORM to prevent SQL injection.
- **Celery Command Injection:**
  - Since we are **not** generating SSL certs via subprocess anymore (using wildcard), this risk is minimized. However, if we ever generate PDFs using `wkhtmltopdf` via subprocess, we must sanitize the filename parameter using `shlex.quote()`.
- **Rate Limit Bypass Protection:**
  - Use `X-Forwarded-For` headers properly. Set `SECURE_PROXY_SSL_HEADER` and `USE_X_FORWARDED_HOST = True` so that throttling sees the real client IP behind the load balancer, not the internal load balancer IP.

---

### Backend Team Sign-Off Checklist (To Leave the Meeting With)

| # | Action Item | Owner | Deadline |
| :--- | :--- | :--- | :--- |
| 1 | Implement Custom JWT Auth class with `last_login_or_pw_change` invalidation check. | Backend Lead | Sprint 1 |
| 2 | Code the SubdomainMiddleware with Redis caching and DB fallback. Ensure `request.school` is available in all views. | Backend Lead | Sprint 1 |
| 3 | Build the Centralized Exception Handler and create the public `error_codes.json` file. | Backend + FE Lead | Sprint 2 |
| 4 | Set up Celery with `celery_once` (or Redis locks) for webhook idempotency. Define the exponential backoff schedule. | Backend Lead | Sprint 2 |
| 5 | Enforce `.only()` / `.defer()` linting rules and integrate `nplusone` to fail CI on N+1 queries. | Backend Lead | Sprint 1 |
| 6 | Configure `python-json-logger` and inject `correlation_id` into all log statements. | DevOps / Backend | Sprint 1 |
| 7 | Define exact rate limits (10/min for subdomain check, 100/min for simulations) in DRF settings. | Backend Lead | Sprint 1 |
| 8 | Create a Security Review checklist (ban raw SQL/subprocess) and add it to the Pull Request template. | Security Officer / CTO | Immediate |
