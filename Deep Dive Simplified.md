# Visual Math Platform Project Structure

---

## Part 1: The Foundation – Python Project Basics

Before we look at the folders, let’s understand the 3 golden rules of Python projects:

1. **The `__init__.py` magic**: Any folder containing an `__init__.py` file becomes a **Python Package**. This means you can `import` it. Inside this file, you can write code to run when the package is first loaded (or just leave it empty to mark it as a package).
2. **The Virtual Environment (`venv`)**: This is a "bubble" around your project. It ensures the Python version and libraries (like Django) used here don't break other Python projects on your computer. You create it (`python -m venv venv`) and activate it. 
3. **The `manage.py` anchor**: When you run a Python script, the interpreter looks for code starting from that file's location. `manage.py` sets up the entire Django environment so your apps can find each other.

Now, let’s walk through the project root.

---

## Part 2: The Root Directory (The Outer Shell)

When you open the project folder, these are the first files you see. They control the *environment* and the *housekeeping*.

| File/Folder | Absolute Beginner Explanation | Why it exists here |
| :--- | :--- | :--- |
| **`.env`** | A hidden text file. It holds **secrets** like passwords, API keys, and the Django secret key. | If you type your password directly into your code, and you upload that code to GitHub, **everyone** sees it. By putting it here, we keep secrets out of the source code. |
| **`.gitignore`** | A list of files/folders that Git (version control) must **ignore**. | We don't want to upload our secrets (`.env`), our virtual environment (`venv/`), or temporary files to GitHub. It keeps the repository light and secure. |
| **`requirements.txt`** | A plain text shopping list of Python libraries with exact version numbers (e.g., `Django==6.0.7`). | When a new developer joins, they run `pip install -r requirements.txt`. It magically installs every library needed to run the project, perfectly matching the setup. |
| **`manage.py`** | A Python script that Django gives you. It is the **control center**. | You use it to run the server (`runserver`), create database tables (`migrate`), create superusers (`createsuperuser`), and run tests. You never edit this file. |
| **`README.md`** | A "cover letter" for your project, written in Markdown. | It explains to anyone (or future you) what this project does and how to set it up. Always update this! |
| **`Dockerfile`** & **`docker-compose.yml`** | Blueprints for **containers**. Think of this as giving the app a specific "box" to live in, along with its own PostgresSQL and Redis boxes. | This ensures the app runs exactly the same on your laptop, your boss's laptop, and the cloud server. (Ignore this initially if you are running things locally without Docker). |

---

## Part 3: The `config/` Folder – The Brain of Django

In standard Django, you usually have a `settings.py` file. In large projects, we upgrade it to a **Package** (a folder with `__init__.py`). 

### How Python Imports Work Here
If we have `config/settings/base.py`, and we want to use it, Python uses dots: `config.settings.base`. Because `config/` and `settings/` both have `__init__.py`, Python treats them as modules.

### Understanding `settings/` (The Control Panel)
Inside the `settings/` folder:

- **`base.py`**: The generic settings. It says things like: *"We are using PostgreSQL. We have apps named schools, users, etc. We are using Redis for caching. We are in the Asia/Kolkata timezone."*
- **`dev.py`**: *"Take everything from base.py (`from .base import *`). Now, turn DEBUG on so I can see detailed error pages. Allow localhost connections."*
- **`prod.py`**: *"Take everything from base.py. Now, turn DEBUG off. Force HTTPS. Send real emails through SMTP. Use a complex password for the database."*

**For a Beginner**: You almost never edit `prod.py` unless you are deploying. You usually run the server using `dev.py` by setting an environment variable: `export DJANGO_SETTINGS_MODULE=config.settings.dev`.

### The `urls.py` (The Map)
This is the "Table of Contents" for your website. When a user visits `your-site.com/admin/`, Django looks at this file to see which "View" (function) should handle that request.

### `wsgi.py` and `asgi.py` (The Plugs)
*   **WSGI**: The standard plug that allows your server (like Nginx/Gunicorn) to talk to Django.
*   **ASGI**: The modern plug that allows WebSockets (for real-time chat or live collaboration). 
*Note: You never run these directly; the server calls them.*

### `celery.py` (The Mail Sorter)
Django handles web requests immediately. But if you need to send a thousand emails or generate a PDF report, the user shouldn't have to wait. **Celery** takes these heavy tasks and puts them into a queue (Redis) to be processed in the background.

---

## Part 4: The `apps/` Folder – The Organs of the Body

Django uses **Apps** to separate features. A "To-Do List" app handles to-do logic; a "Forum" app handles forum logic. 
We put **all** our custom apps inside a master folder called `apps/`.

**Crucial Beginner Note**: 
Django usually expects app names as strings, e.g., `'schools'`. But we have a folder called `apps`, so Python needs the full path: `'apps.schools'`. 
If we look at `settings/base.py`, we see `'apps.schools'` in the `INSTALLED_APPS` list. This tells Django to look *inside* the `apps/` folder.

### `common/` (The Utility Drawer)
This isn't a feature; it's a toolbox.
- **`models.py`**: Contains `TimestampedModel`. This is an *abstract* model. You never create a table for it directly. Instead, other models *inherit* from it, automatically getting `created_at` and `updated_at` fields without you writing them repeatedly.
- **`mixins.py`**: A "Mixin" is a tiny piece of logic you can sprinkle into other classes. For example, a mixin that automatically adds the current tenant to a database query.

### `schools/` (The Tenant Manager)
- **`models.py`**: Defines the `School` class. It has fields like `subdomain`, `logo`, and `primary_color`.
- **`middleware.py`**: **This is the Front Door Security Guard.** Every time a user makes a request, this runs *first*. It looks at the subdomain (e.g., `schoola.platform.com`), finds the matching School in the database, and attaches it to the current request (`request.tenant`). 
- **`views.py`**: The logic for the school's homepage and dashboard.

### `users/` (Who is logging in?)
Django has a default `User` model. We override it here.
- **`models.py`**: We add `role` (Principal, Teacher, Student). We link it to the `School` model. This means a user belongs to exactly ONE school.
- **Why this file matters**: When a student logs in, the custom auth middleware checks if `logged_in_user.school_id` matches `request.tenant.id`. If not, they get kicked out.

### `curricula/`, `simulations/`, `assessments/` (The Content)
- **`curricula/`**: Defines the hierarchy: Curriculum (Class 8 Maths) -> Chapter (Algebra) -> Lesson (Linear Equations).
- **`simulations/`**: Stores the technical configuration for the interactive graphs (e.g., "load p5.js library, show a graph of y=mx+c, allow the user to drag the slope slider").
- **`assessments/`**: Stores quizzes, student answers, and grades.

### `payments/` (The Money)
- **`webhooks.py`**: Razorpay (payment gateway) sends a silent HTTP request to this file when a payment is successful. We don't trust the frontend to tell us a payment is successful; we wait for Razorpay's server to tell us. This is called a Webhook.

### `analytics/` (The Dashboard)
Will contain logic to calculate how many students passed a quiz, or which lessons are the hardest.

---

## Part 5: The Supporting Folders

| Folder | Purpose for a Beginner |
| :--- | :--- |
| **`static/`** | **Hard-coded assets** (CSS, vanilla JavaScript, images). If you write a logo that is the same for all schools, put it here. During production, Django collects all these into one place (`collectstatic`). |
| **`media/`** | **User-uploaded assets**. When a school principal uploads their *custom logo*, it goes here. When a student uploads an assignment, it goes here. This folder grows over time and is backed up regularly. |
| **`templates/`** | **HTML files**. These are the "skeletons" of the web pages. `base.html` is the master skeleton. Other pages "extend" it, filling in the content blocks. This uses Django Template Language (DTL). |
| **`middleware/`** | **Global request filters**. While `schools/middleware.py` handles tenants, this top-level `middleware/` handles cross-cutting concerns like logging every request's speed, or checking if an IP is banned. |
| **`libs/`** | **Third-party adapters**. Suppose Razorpay updates their API. You only need to change the code in `libs/razorpay/client.py` to fix it. The rest of your `payments/` app stays untouched. |
| **`scripts/`** | **Cron jobs / maintenance**. You run `python manage.py seed_db` to populate your local database with fake testing data, so you don't have to manually create a school every time. |

---

## Part 6: A Beginner's Walkthrough – What happens when the server starts?

Let's simulate you typing `python manage.py runserver`.

1. **Boot-up**: Python reads `manage.py`. It sees `DJANGO_SETTINGS_MODULE` is set (say, to `config.settings.dev`).
2. **Config Load**: It opens `config/settings/dev.py`. This file says `from .base import *`. So it loads all the base settings first, then overrides DEBUG to `True`. 
3. **App Registration**: It reads `INSTALLED_APPS` in `base.py` and finds `'apps.schools'`, `'apps.users'`, etc. It "registers" them. 
4. **URL Mapping**: It loads `config/urls.py` and sees the routes.
5. **Server Listen**: The server starts on `localhost:8000`.

**Now, a student visits `schoola.localhost:8000/lesson/1/`.**

1. **Middleware (Tenant)**: 
   - `request.get_host()` returns `schoola.localhost`.
   - `middleware/tenant.py` splits this to get `schoola`.
   - It queries the database: "Is there a school with subdomain `schoola`?" Yes! It switches the PostgreSQL connection to the *specific database schema* for `schoola` (so we only see their data).
   - It attaches `school` to the request.

2. **Middleware (Auth)**: 
   - The user isn't logged in yet, but they are served the public lesson.
   - The `Lesson` model query happens. Because the Tenant middleware changed the database schema, Django automatically searches `schoola`'s private data. It finds **Lesson 1** defined for that school.

3. **View Logic** (`curricula/views.py`): 
   - Fetches the lesson content.
   - Fetches the simulation config from the `simulations` app.

4. **Template Rendering**:
   - The view loads `templates/lesson_detail.html`.
   - It passes the `school` object. The HTML uses `{{ school.primary_color }}` to change the header color dynamically.
   - It passes the `simulation_config` as a JavaScript variable.

5. **HTML Delivery**: The browser receives the HTML, loads the CSS from `static/css/`, and runs the vanilla JavaScript. The JavaScript uses the config to draw a p5.js interactive graph on the screen.

---

## Part 7: Critical Python Concepts Applied in this Structure

### 1. What is `__init__.py` for?
Inside `apps/__init__.py` (and all others), you see an empty file. Why? 
Without this file, Python does not allow you to write `import apps.schools`. The `__init__.py` tells Python: *"This folder is a module, treat it as part of the code."*

### 2. Relative Imports in Settings
Look at `config/settings/dev.py`:
```python
from .base import *  # The dot (.) means "in the same directory as me".
```
This is a Python relative import. It tells Python to go up one level to the `settings` package and grab everything from `base.py`.

### 3. Python Paths
When you write `from apps.schools.models import School`, Python looks for the `apps` folder. 
*Where does it find it?* 
In our `manage.py`, there is a line:
```python
sys.path.append(str(BASE_DIR / 'apps'))
```
This is a **hack** but a common one. It modifies Python's search path to include the `apps/` directory. Alternatively, we could just use `import schools.models` if we added `apps/` to the path. In our settings, we consistently use `apps.schools`, so we must ensure the root directory (`BASE_DIR`) is in the path (which `manage.py` automatically handles). 

### 4. The `__all__` variable in `config/__init__.py`
You might see:
```python
from .celery import app as celery_app
__all__ = ('celery_app',)
```
This tells Python: "If someone imports `*` from config, only import `celery_app`". It is a good security practice to hide unnecessary internals.

---

## Part 8: Common Beginner Pitfalls (and how this structure avoids them)

1. **Pitfall**: Accidentally running `DEBUG=True` on the live server, exposing errors to hackers.
   - **The Fix**: Our `prod.py` explicitly sets `DEBUG = False`. If you forget to set the environment variable, Django fails to find `prod.py` and won't start easily, forcing you to check it.
2. **Pitfall**: Forgetting to run database migrations for a new app.
   - **The Fix**: Because all apps are in `INSTALLED_APPS`, `python manage.py makemigrations` automatically scans `apps/` and finds the new models.
3. **Pitfall**: Uploading a file to `media/` and losing it when you redeploy.
   - **The Fix**: We know `media/` is separated. In production, we mount `media/` to a persistent cloud storage (S3) or an external drive, keeping it safe.
4. **Pitfall**: Hard-coding the Razorpay API key in the `payments/views.py`.
   - **The Fix**: We use the `libs/razorpay/client.py` which reads from the `.env` file. The API key never appears in the code view.

---

## Part 9: Summary – The Mindset Shift

As an absolute beginner, **do not try to understand every line of code at first**. Instead, understand the **flow of files**:

- **Config** is the Brain (startup logic).
- **Apps** are the Organs (features).
- **Static/Media/Templates** are the Skin (what the user sees).
- **Libs** are the Hands (touching external services).
- **Scripts** are the Maintenance Crew.

When you want to add a new feature (e.g., "Homework Submission"):
1. Go to `apps/`.
2. `python manage.py startapp homework apps/homework`.
3. Add `apps.homework` to `INSTALLED_APPS` in `settings/base.py`.
4. Define your models in `apps/homework/models.py`.
5. Create views, URLs, and templates.
6. Run `makemigrations` and `migrate`.

Because the Tenant Middleware runs globally, your new "Homework" app automatically inherits the subdomain isolation without writing a single line of code for tenants. 

This structure is designed to let you **focus on math simulations** and **school management**, not waste time wrestling with configuration.
