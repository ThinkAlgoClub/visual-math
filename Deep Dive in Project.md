# Visual Math Platform Project Structure  

---

## 1. Why This Structure?

The project layout is **not arbitrary** – it is designed to support:

- **Multi‑tenancy** – Each school operates in isolation, with its own data schema and subdomain.
- **White‑labelling** – Schools can customise colours, logos, and even content.
- **Scalability** – PostgreSQL schemas, Redis caching, Celery background tasks, and a modular Django design allow horizontal scaling.
- **Security** – Environment variables, strict settings separation, and middleware layers enforce tenant isolation and authentication.
- **Maintainability** – Clear separation of concerns: each app handles one domain, configuration is centralised, and custom code lives in predictable places.

This structure follows Django’s “loose coupling, high cohesion” principle and is battle‑tested in production‑grade SaaS applications.

---

## 2. Root Files – The Foundation

### `.env` – The Vault of Secrets
- **Purpose**: Store all environment‑specific variables (secret keys, database credentials, API keys) **outside** the codebase.
- **Best Practice**: Never commit it. Use `.env.example` as a template for other developers.
- **Key variables** (examples):
  ```env
  SECRET_KEY=django-insecure-xyz
  DEBUG=True
  DB_NAME=visual_math_db
  DB_USER=app_user
  DB_PASSWORD=supersecret
  REDIS_URL=redis://localhost:6379/1
  RAZORPAY_KEY_ID=rzp_test_xxx
  RAZORPAY_KEY_SECRET=test_secret_yyy
  ```
- **Loading**: In `settings/base.py`, we use `python-dotenv` to load these into `os.environ`.

---

### `.gitignore` – Protect Your Project
- **Must‑ignore**:
  - Python: `venv/`, `*.pyc`, `__pycache__/`
  - Django: `*.sqlite3`, `media/`, `staticfiles/`
  - Secrets: `.env`
  - IDE: `.vscode/`, `.idea/`
- **Why**: Avoid accidental exposure of secrets, local databases, and virtual environments.

---

### `docker-compose.yml` & `Dockerfile` – Containerisation
- **Dockerfile**: Builds a slim Python image, installs dependencies, and copies the code.
- **docker‑compose.yml**: Defines services – **web** (Django), **db** (PostgreSQL), **redis** (cache/broker).  
  - Environment variables are passed from `.env`.
  - Volumes persist data (PostgreSQL, media files).
  - Ports mapped for local development.
- **Insight**: This setup mirrors production – you can test multi‑service interactions locally.

---

### `manage.py` – Django’s Swiss Army Knife
- Runs the development server, creates migrations, runs tests, etc.
- **Pro tip**: Use `python manage.py check --deploy` before production to catch security issues.

---

### `requirements.txt` – Dependencies with Versions
- Pins every package to a specific version to ensure consistency.
- Includes production and development dependencies (e.g., `pytest`, `coverage` for testing).
- **Insight**: For larger projects, split into `requirements/base.txt`, `dev.txt`, `prod.txt` and use `-r` includes.

---

### `README.md` – The Project’s Face
- Should contain: project description, setup steps, tech stack, and contact info.
- For schools/white‑labelling, mention how to configure a new tenant.

---

## 3. `config/` – The Nervous System

### `settings/` Package – Environment‑Aware Configuration
Instead of a monolithic `settings.py`, we use **composition**:

- **`base.py`**: Common settings for **all** environments.  
  - Installed apps, middleware, database (defaults), Redis, Celery, authentication, static/media, internationalisation.
  - **Crucial**: The `AUTH_USER_MODEL` is set to `'users.User'` – we customise the user model early.
- **`dev.py`**: Imports everything from `base` and overrides:  
  - `DEBUG=True`, `ALLOWED_HOSTS=['localhost']`
  - Email backend to `console.EmailBackend` (prints emails to terminal).
  - Optional: `django-debug-toolbar` for performance insights.
- **`prod.py`**: Overrides for production:  
  - `DEBUG=False`, strict `ALLOWED_HOSTS` from env.
  - `SECURE_SSL_REDIRECT=True`, cookie security flags.
  - Static file storage with `ManifestStaticFilesStorage` for cache‑busting.
  - Email via SMTP (e.g., SendGrid).
  - Database connection pooling (`CONN_MAX_AGE`).
- **`staging.py`**: A copy of `prod` but with `DEBUG` maybe `True` on a test server.

**Why this is brilliant**:
- No code duplication – only overrides.
- No risk of accidentally leaving debug mode on in production.
- CI/CD can set `DJANGO_SETTINGS_MODULE` per environment.

---

### `urls.py` – The Request Router
- Root URLconf includes:
  - `/admin/` – Django admin (only for super admins).
  - `/api/` – REST API endpoints (we will create in Lesson 12).
  - `''` – For the main frontend (served by `apps.schools.views.home`).
- In development, we also serve `media/` and `static/` via `static()` helper.

---

### `wsgi.py` & `asgi.py` – Entry Points for Servers
- **WSGI**: Used by Gunicorn (production) – standard synchronous requests.
- **ASGI**: Used for async capabilities (WebSockets, long polling) – we may use it later for real‑time simulation collaboration.

---

### `celery.py` – Background Tasks Made Easy
- Creates a Celery app with the Django settings module.
- Auto‑discovers tasks in all installed apps.
- **Why separate?** Keeps Celery configuration out of `settings.py`, making both files cleaner.

---

## 4. `apps/` – The Business Logic

Each app is a **self‑contained module** with its own models, views, serializers, etc.  
They are all placed under `apps/` to keep the project root clean – this is a convention that many large Django projects follow.

### `common/` – The Utility Belt
- **`models.py`**: Contains **abstract base models** like `TimestampedModel` (adds `created_at`/`updated_at`) and `SoftDeleteModel` (adds `is_deleted`/`deleted_at`).  
  *Insight*: Soft deletion is critical in schools – we may want to restore a accidentally deleted student record.
- **`managers.py`**: Custom model managers (e.g., `ActiveManager` that filters out soft‑deleted records).
- **`mixins.py`**: Mixin classes for views (e.g., `TenantAwareMixin` that filters querysets by the current school).
- **`validators.py`**: Reusable validators (e.g., `validate_phone_number`, `validate_pincode`).

---

### `schools/` – The Tenant Engine
- **`models.py`**: The `School` model is the **central tenant**.
  - Contains all white‑labelling fields: `primary_color`, `secondary_color`, `logo`, `favicon`.
  - Tracks subscription: `subscription_plan`, `subscription_end_date`.
  - **Method** `is_subscription_active` checks if the school’s subscription is current – used in middleware to block access if expired.
- **`middleware.py`**: The `TenantMiddleware` is **the gatekeeper**.
  - Extracts subdomain from `request.get_host()`.
  - Queries `School` by subdomain.
  - Sets `request.tenant` and then **switches PostgreSQL schema** to `tenant_{school_id}` via `SET search_path`.
  - If no school found or subscription expired, raises 404 or redirects to a payment page.
  - **Crucial**: This runs **before** authentication, so even unauthenticated users see the correct branding.
- **`serializers.py`**: For DRF – serialises `School` data (but we rarely expose full school details via API, only admin).
- **`urls.py`**: Handles the home page and dashboard views for the tenant.

---

### `users/` – Identity & Access Control
- **`models.py`**: Extends Django’s `AbstractUser`.
  - Adds `role` (Principal, Teacher, Student, Parent, Admin).
  - Links to `School` (foreign key) – every user belongs to a tenant.
  - Additional fields: `phone`, `date_of_birth`, `class_grade`, `section` (for students).
  - **`email` and `school` are unique together** – a user can have the same email across schools, but not within the same school.
- **`admin.py`**: Custom admin interface – list filters by role/school, search by username/email.
- **`forms.py`**: Custom forms for registration, profile update, etc., that include role selection and school assignment (for principals).
- **`permissions.py`** (not in the list, but you can add): Custom permission classes for DRF – e.g., `IsPrincipal`, `IsTeacher`, `IsStudent`.
- **Insight**: We use Django’s built‑in authentication backends, but we can add additional checks in middleware to ensure a user’s school matches the tenant.

---

### `curricula/` – Content Hierarchy
- Models: `Curriculum` (class/board), `Chapter`, `Lesson`.
- Each `Lesson` has a `simulation_config` JSON field – stores parameters for the interactive simulation (e.g., x‑axis range, number of objects, etc.).
- Content is stored as HTML or Markdown in the `content` field.
- The `order` field ensures proper sequencing.
- **Why separate from `simulations`?** A lesson may have multiple simulations, or we may want to swap simulation libraries without changing lesson content. The `Simulation` model in the `simulations` app links one‑to‑one with `Lesson`.

---

### `simulations/` – The Visual Engine
- **`models.py`**: `Simulation` has `library` (p5, three, mathjax), `script_url` (CDN or local path), and a JSON `config`.
- **`views.py`**: Probably an API endpoint that returns the simulation configuration, or a view that renders the simulation container.
- **Insight**: This app is **frontend‑friendly** – it will generate the required JavaScript to load the simulation. The frontend will call `/api/simulation/<lesson_id>/` to get the configuration.

---

### `assessments/` – Tracking Learning
- Models for quizzes, questions, student attempts, and progress.
- We’ll design this in later lessons – but the app is already here to house those models.

---

### `payments/` – Revenue Engine
- **`models.py`**: `Subscription` links to `School` and stores Razorpay subscription ID, plan, amount, dates, and last payment ID.
- **`razorpay_client.py`**: Wraps Razorpay SDK calls – creates orders, fetches payments, handles webhook verification.
- **`webhooks.py`**: Contains a Django view that receives POST requests from Razorpay.
  - Verifies the signature using Razorpay’s secret.
  - Processes events: `payment.captured`, `subscription.charged`, `subscription.cancelled`.
  - Updates `Subscription` status and sends emails via Celery tasks.
- **Why separate?** Keeps webhook handling isolated and idempotent.

---

### `analytics/` – Insights for Principals
- Models for aggregated data (e.g., daily active users, lesson completion rates).
- Views for dashboards (DRF or HTML).
- Will use Celery to pre‑compute reports daily.

---

## 5. `static/`, `media/`, `templates/` – Presentation Layer

### `static/` – Global Assets
- Contains CSS, JS, images used across **all** tenants.
- The `.css` files will use CSS custom properties (variables) that get overridden by tenant colours – this enables white‑labelling without rebuilding assets.
- JavaScript is modular (as described in Lesson 2) – we keep it in `js/modules/`.
- **Collecting**: `python manage.py collectstatic` gathers all static files into a single directory for production.

### `media/` – User Uploads
- Tenant‑specific subfolders: `media/tenant_{school_id}/logos/`, `media/tenant_{school_id}/thumbnails/`.
- The `School` model’s `logo` and `favicon` fields use `upload_to` that includes the tenant ID (we’ll override `upload_to` dynamically).

### `templates/` – HTML Skeletons
- `base.html` defines the layout (navbar, footer, content blocks).
- Uses template tags to inject tenant‑specific data (e.g., `{{ request.tenant.primary_color }}`).
- We will also have tenant‑specific templates if needed (e.g., for custom dashboards), but we can reuse the same base template with dynamic styles.

---

## 6. `middleware/` – The Request Interceptors

Custom middleware classes that are registered in `MIDDLEWARE` in the order of execution.

### `tenant.py`
- **Execution**: Early in the pipeline (before authentication).
- **Logic**:
  1. Parse host to get subdomain.
  2. Fetch `School` from cache (Redis) to avoid DB hit on every request.
  3. If found, set `request.tenant` and switch DB schema.
  4. If not, return 404 (or redirect to a landing page).
  5. Check subscription status – if expired, redirect to `/payment/`.
- **Insight**: Caching the tenant lookup reduces database load significantly.

### `auth.py`
- **Execution**: After authentication.
- **Checks**: That the authenticated user belongs to the current tenant (by comparing `user.school_id` with `request.tenant.id`).
- If not, logs them out and returns a 403.

### `logging.py`
- Logs request details (method, path, status, duration) for monitoring.
- Could also log security events (failed logins, unauthorised access attempts).

---

## 7. `libs/` – External Service Adapters

We keep third‑party SDK wrappers here to centralise configuration and error handling.

### `razorpay/client.py`
- Initialises the Razorpay client with key/secret from env.
- Provides convenience methods: `create_order()`, `verify_payment_signature()`, `get_subscription()`.
- **Why not in `payments/`?** Keeps the payments app focused on models and business logic; the library wrapper is a separate concern.

---

## 8. `scripts/` – Maintenance Commands

### `seed_db.py`
- A Django management command (you run it with `python manage.py seed_db`).
- Creates a demo school and a superuser (principal) for quick testing.
- **Insight**: We can also add commands to generate dummy lessons, simulations, and student data for performance testing.

---

## 9. How It All Works Together – A Request Journey

1. A student visits `mygovtgirls.platform.com/lesson/5/`.
2. **DNS** resolves to the load balancer.
3. **Nginx** (or CDN) passes the request to Gunicorn.
4. **TenantMiddleware**:
   - Extracts `mygovtgirls` as subdomain.
   - Fetches `School` from Redis (or DB).
   - Sets `request.tenant` and switches PostgreSQL schema to `tenant_123`.
5. **AuthenticationMiddleware**: Validates the session; if the student is logged in, the user object is attached.
6. **AuthMiddleware** (custom): Checks `user.school_id == request.tenant.id`.
7. **URLResolver**: Matches `/lesson/5/` – routes to the `curricula` app view.
8. **View**:
   - Fetches the `Lesson` with ID 5 from the tenant schema.
   - Fetches the associated `Simulation` (if any).
   - Renders a template with lesson content and a container for the simulation.
9. **Template** renders using the school’s theme (colours from `request.tenant`).
10. **JavaScript** on the page:
    - Loads the simulation library (p5.js) from CDN.
    - Makes an AJAX call to `/api/simulation/5/` to get the configuration.
    - Initialises the simulation.
11. As the student interacts, progress is saved via API calls to the backend (which, of course, go through the same tenant middleware).

---

## 10. Extending the Structure – Adding a New Feature

Suppose we want to add a “Forum” where students can ask questions.

1. **Create a new app**: `python manage.py startapp forum apps/forum`
2. **Define models**: `Question`, `Answer`, `Upvote` – all with `school` foreign key.
3. **Register in `INSTALLED_APPS`** (in `base.py`).
4. **Add URLs** in `config/urls.py` (e.g., `path('forum/', include('apps.forum.urls'))`).
5. **Create serializers, views, templates** – they will automatically inherit the tenant middleware and schema switching.
6. **Add tests** in `apps/forum/tests.py`.

Because the tenant isolation is handled at the middleware/schema level, the new app **immediately** becomes multi‑tenant without any extra work.

---

## 11. Deployment & Scalability Considerations

- **Horizontal Scaling**: Multiple Gunicorn instances behind a load balancer.  
  - Redis is shared – sessions and cache are centralised.
  - PostgreSQL uses schemas – no conflict across app servers.
- **Database Optimisation**:
  - Index foreign keys (especially `school_id` – though we don’t store it in every table because of schemas, we still need indexes on frequently queried columns).
  - Use connection pooling (e.g., PgBouncer).
- **Cache Strategy**:
  - Use Redis for **tenant lookup** (subdomain → school ID), **lesson content**, **simulation configs**.
  - Invalidate cache on model save using `post_save` signals.
- **Media Storage**:
  - In production, use Amazon S3 or DigitalOcean Spaces to serve media files – the `MEDIA_URL` is configured accordingly.
- **White‑Labelling**:
  - All branding (logo, colours) is stored in the `School` model. The frontend applies these via CSS variables.
  - For full white‑label (custom domain), we can map a school’s custom domain to the same application by adding it to `ALLOWED_HOSTS` and updating the tenant middleware to check both subdomain and custom domain.

---

## 12. Security Hardening Built into the Structure

- **Settings Separation**: Prevents debug mode in production.
- **Environment Variables**: Secrets never hard‑coded.
- **Multi‑Tenancy Isolation**: PostgreSQL schemas are the strongest isolation – even if a query bypasses a `tenant_id` filter, the data is physically separate.
- **Middleware Checks**: Ensure user belongs to the tenant.
- **CSRF & XSS Protection**: Django’s defaults are enabled.
- **Webhook Verification**: Razorpay signature check prevents fake payment events.
- **User Input Validation**: DRF serializers and Django forms validate all input.

---

## 13. Conclusion

This project structure is **deliberate and robust**. It gives you:

- **Clear separation of concerns** – each app does one thing well.
- **Environment‑aware settings** – safe and flexible.
- **Multi‑tenancy at the schema level** – scalable and secure.
- **Extensibility** – adding new features is straightforward.
- **Production‑readiness** – caching, background tasks, and static/media handling are built in.
