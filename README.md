# Visual Math Platform – Project Structure Guide  

**Every file and folder** in the `visual-math-platform/` project structure.  

---

## The Big Picture

```
visual-math-platform/
├── .env
├── .gitignore
├── docker-compose.yml
├── Dockerfile
├── manage.py
├── requirements.txt
├── README.md
│
├── config/
├── apps/
├── static/
├── media/
├── templates/
├── middleware/
├── libs/
└── scripts/
```

This is a **Django web application** with additional services (PostgreSQL, Redis, Celery) for a multi‑tenant, interactive mathematics platform.

---

## Root Files (Top Level)

### `.env`
**What**: A hidden file (not shared in Git) that stores **environment variables**.  
**Why**: Keeps secrets (passwords, API keys) out of your code.  
**Example contents**:
```
SECRET_KEY=your-secret-key
DB_PASSWORD=supersecret
RAZORPAY_KEY_ID=test_key
```
**For beginners**: Never commit this file to Git! Add it to `.gitignore`.

---

### `.gitignore`
**What**: A list of files/folders that Git should ignore.  
**Why**: Prevents committing temporary files, environment secrets, compiled files, etc.  
**Typical ignores**:
```
venv/
*.pyc
__pycache__/
.env
media/
staticfiles/
```

---

### `docker-compose.yml`
**What**: Defines how to run **multiple containers** (Django, PostgreSQL, Redis) together.  
**Why**: Simplifies running the whole system with one command (`docker-compose up`).  
**For beginners**: If you don’t use Docker, you can ignore this file and run services manually.

---

### `Dockerfile`
**What**: Instructions to build a Docker image for the Django app.  
**Why**: Makes the app portable and reproducible.

---

### `manage.py`
**What**: Django’s command‑line utility.  
**Why**: You use it to run the server, create migrations, create superusers, etc.  
**Example**: `python manage.py runserver`

---

### `requirements.txt`
**What**: Lists all Python packages needed for the project (with versions).  
**Why**: Install everything with `pip install -r requirements.txt`.

---

### `README.md`
**What**: A descriptive file for anyone who views the project on GitHub.  
**Why**: Explains what the project does, how to set it up, and where to find docs.

---

## `config/` – Django Project Settings

This folder holds the **core Django configuration**. It’s the heart of your project.

| File/Folder | Purpose |
|-------------|---------|
| `__init__.py` | Marks `config/` as a Python package. |
| `asgi.py` | Entry point for ASGI servers (for WebSockets, async). |
| `wsgi.py` | Entry point for WSGI servers (like Gunicorn) – used in production. |
| `urls.py` | The root URL dispatcher. It maps URLs to views. |
| `celery.py` | Configures Celery for background tasks (e.g., sending emails). |
| `settings/` | A **package** containing different settings files. |

### Inside `settings/`

- **`base.py`** – Shared settings for all environments (databases, installed apps, middleware, time zone, etc.).
- **`dev.py`** – Overrides for development: `DEBUG=True`, local hosts, console email.
- **`prod.py`** – Overrides for production: `DEBUG=False`, HTTPS, remote email, stricter security.
- **`staging.py`** – Optional for a staging server.

**Why split?**  
So you can have different configurations for your local machine, testing server, and live site without changing the core code.

**How to use**: Set environment variable `DJANGO_SETTINGS_MODULE=config.settings.dev` (or `prod`).

---

## `apps/` – All Your Django Applications

Django apps are reusable modules. Here we keep all our custom apps inside `apps/` to keep the project tidy.

### `common/`
**Purpose**: Shared utilities used across many apps.  
**Files**:
- `models.py` – Abstract base models (e.g., `TimestampedModel` with `created_at`/`updated_at`).
- `managers.py` – Custom model managers for common queries.
- `mixins.py` – Mixin classes to reuse in other models/views.
- `validators.py` – Custom validation functions (e.g., for phone numbers).

### `schools/`
**Purpose**: Manages **tenants** – each school that uses the platform.  
**Key model**: `School` – stores name, subdomain, branding, subscription status.  
**Middleware**: Will contain `TenantMiddleware` to detect the school from the subdomain.

### `users/`
**Purpose**: Custom user model and role management.  
**Key model**: `User` – extends Django’s `AbstractUser` to add `role` (principal, teacher, student) and link to a `School`.

### `curricula/`
**Purpose**: Organises the educational content.  
**Models**: `Curriculum`, `Chapter`, `Lesson` – nested structure for class/board/subject.

### `simulations/`
**Purpose**: Stores configurations for interactive math simulations.  
**Model**: `Simulation` – links to a `Lesson` and stores which library to use (`p5.js`, `Three.js`, etc.) and its parameters.

### `assessments/`
**Purpose**: Manages quizzes, assignments, and student progress.  
**Future**: Models for questions, student responses, and scores.

### `payments/`
**Purpose**: Integrates with **Razorpay** for subscriptions.  
**Files**:
- `models.py` – `Subscription` model linked to a school.
- `webhooks.py` – Handles incoming webhooks from Razorpay (payment events).
- `razorpay_client.py` – Wrapper for Razorpay API calls.

### `analytics/`
**Purpose**: Generates reports and dashboards for principals and teachers.  
**Future**: Aggregated progress data, usage statistics.

---

## `static/` – Global Static Files

Files that are served directly to the browser (CSS, JavaScript, images).  
They are **not** user‑uploaded.

- `css/` – Stylesheets.
- `js/` – All your JavaScript (no React – vanilla JS).
- `images/` – Icons, logos, backgrounds.

These files are collected into one folder during deployment using `collectstatic`.

---

## `media/` – User‑Uploaded Files

This folder stores files uploaded by users (e.g., school logos, student avatars, simulation thumbnails).  
In production, this is often stored on a cloud service (S3), but locally it’s kept here.

---

## `templates/` – Global HTML Templates

Contains base HTML files that other pages extend.  
- `base.html` – The main layout with header, footer, and blocks for content.
- `404.html`, `500.html` – Custom error pages.

---

## `middleware/` – Custom Django Middleware

Middleware runs on every request/response. Here you add custom logic.

| File | Purpose |
|------|---------|
| `tenant.py` | Detects the subdomain, finds the matching `School`, and sets the database schema for that tenant. |
| `auth.py` | Additional authentication checks (e.g., ensuring the user belongs to the correct school). |
| `logging.py` | Logs requests for debugging/auditing. |

---

## `libs/` – Third‑Party Integration Wrappers

Keeps code that talks to external services separate.  
- `razorpay/client.py` – Initialises the Razorpay client using keys from environment variables.

---

## `scripts/` – One‑Off Scripts

For management commands or cron jobs.  
- `seed_db.py` – A script that populates the database with dummy data for testing (school, principal user, etc.).

---

## How to Run the Project (Step‑by‑Step for Beginners)

1. **Clone the repository** (or create the structure).  
2. **Set up a virtual environment**  
   ```bash
   python3 -m venv venv
   source venv/bin/activate   # On Windows: venv\Scripts\activate
   ```
3. **Install dependencies**  
   ```bash
   pip install -r requirements.txt
   ```
4. **Create a `.env` file** in the root with your own secret key and database credentials (copy from example).  
5. **Start PostgreSQL and Redis** (or use Docker).  
6. **Run migrations**  
   ```bash
   python manage.py migrate
   ```
7. **Create a superuser**  
   ```bash
   python manage.py createsuperuser
   ```
8. **Run the development server**  
   ```bash
   python manage.py runserver
   ```
9. Visit `http://localhost:8000/admin/` to log in.

---

## Why This Structure?

- **Modularity** – Each app handles one domain, making it easy to add/remove features.
- **Scalability** – With multi‑tenancy (PostgreSQL schemas) and caching (Redis), the platform can grow.
- **Security** – Environment variables keep secrets safe; settings split avoids accidental debug in production.
- **Developer‑friendly** – Clear separation of static/media/templates; custom middleware and libs make the code readable.

