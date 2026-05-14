# Django Polls — Architecture & Developer Guide

## What This Is

A minimal Django tutorial application implementing a polling system. Users can view questions, vote on choices, and see results. It covers the canonical Django pillars: models, views, templates, URLs, admin, static files, and tests.

---

## Project Layout

```
django-polls/
├── manage.py               # CLI entry point for all Django commands
├── mysite/                 # Project package (config, not app logic)
│   ├── settings.py         # All configuration
│   ├── urls.py             # Root URL dispatcher
│   └── wsgi.py             # Production WSGI entry point
├── polls/                  # The only Django app
│   ├── models.py           # Question + Choice data models
│   ├── views.py            # IndexView, DetailView, ResultsView, vote()
│   ├── urls.py             # App-level URL patterns
│   ├── admin.py            # Admin panel customisation
│   ├── tests.py            # Model + view test suite
│   ├── apps.py             # App registry (PollsConfig)
│   ├── migrations/         # Auto-generated schema history
│   ├── templates/polls/    # HTML templates (index, detail, results)
│   └── static/polls/       # CSS + images
└── templates/admin/        # Project-level admin template overrides
    └── base_site.html      # Custom "Polls Administration" branding
```

---

## Component Breakdown

### `mysite/` — Project Config Layer

This package owns no business logic. Its job is to wire everything together.

| File | Purpose |
|------|---------|
| `settings.py` | All knobs: DB, installed apps, middleware, templates, static files |
| `urls.py` | Mounts `polls/` at `/polls/` and the admin at `/admin/` |
| `wsgi.py` | Exposes the `application` object for WSGI servers (gunicorn, uWSGI) |

### `polls/models.py` — Data Layer

Two models, one relationship:

```
Question ──< Choice
```

- **Question**: holds `question_text` and `pub_date`. The `was_published_recently()` method is annotated for the admin (sortable, boolean icon, human label).
- **Choice**: belongs to a Question via a `CASCADE` FK; stores `choice_text` and `votes`.

The temporal filtering logic (`pub_date <= now`) lives in view querysets, not on the model itself — see *Gotchas* below.

### `polls/views.py` — Business Logic Layer

| View | Type | Route | Purpose |
|------|------|-------|---------|
| `IndexView` | `ListView` | `/polls/` | Last 5 published questions |
| `DetailView` | `DetailView` | `/polls/<pk>/` | One question + radio choices |
| `ResultsView` | `DetailView` | `/polls/<pk>/results/` | One question + vote counts |
| `vote()` | function | `/polls/<pk>/vote/` | POST handler; increments vote, redirects |

Both `DetailView` and `ResultsView` override `get_queryset()` to restrict to published questions, so requesting an unpublished question's URL returns a 404.

### `polls/urls.py` — Routing Layer

Sets `app_name = 'polls'` for namespace isolation. All four URL patterns use named routes (`index`, `detail`, `results`, `vote`), which templates reference as `{% url 'polls:detail' question.id %}`.

### `polls/admin.py` — Admin Layer

- `ChoiceInline` embeds choices directly inside the Question edit form (3 extra blank rows).
- `QuestionAdmin` configures list columns, sidebar filter, search box, and a fieldset that collapses the date field.
- Custom admin branding via `templates/admin/base_site.html`.

### `polls/tests.py` — Test Layer

Tests are split into two classes:

- `QuestionModelTests`: unit-tests `was_published_recently()` across past, present, and future dates.
- `QuestionIndexViewTests` / `QuestionDetailViewTests`: integration tests via Django's test `Client`, verifying temporal filtering and correct HTTP responses.

A shared `create_question()` factory helper reduces boilerplate.

---

## Data Flow — Voting

```
Browser
  └─ GET  /polls/                   → IndexView  → index.html (question list)
  └─ GET  /polls/<pk>/              → DetailView → detail.html (radio form)
  └─ POST /polls/<pk>/vote/         → vote()
       ├─ choice found  → Choice.votes += 1, redirect → /polls/<pk>/results/
       └─ choice missing → re-render detail.html with error message
  └─ GET  /polls/<pk>/results/      → ResultsView → results.html
```

---

## Gotchas

### 1. Race condition in `vote()`

```python
# polls/views.py
selected_choice.votes += 1
selected_choice.save()
```

This is a read-modify-write. Two simultaneous votes on the same choice will silently drop one. Fix with an atomic increment:

```python
from django.db.models import F
selected_choice.votes = F('votes') + 1
selected_choice.save()
```

### 2. `DEBUG = True` and `SECRET_KEY` is a placeholder

`settings.py` ships with `DEBUG = True` and a dummy secret key. Both must be changed before any real deployment. `ALLOWED_HOSTS` is also empty, which accepts any hostname.

### 3. SQLite in production

`DATABASES` points to a local `db.sqlite3` file. SQLite has no connection pooling and poor write concurrency. Fine for a tutorial; swap to PostgreSQL before any real traffic.

### 4. No MEDIA handling

There's `STATIC_URL` but no `MEDIA_URL` / `MEDIA_ROOT`. If you add file uploads to questions or choices, you'll need to configure this and serve media files separately in production.

### 5. Temporal filtering is duplicated

`IndexView`, `DetailView`, and `ResultsView` each independently filter `pub_date__lte=timezone.now()`. If you add a third state (e.g. "archived"), you have to update every queryset.

### 6. No pagination

`IndexView` hard-codes `.order_by('-pub_date')[:5]`. With more than 5 questions the rest are simply invisible — there's no pagination or "load more".

### 7. Django 2.1 is EOL

`requirements.txt` pins Django 2.1. This version has been end-of-life since April 2019 and has known security issues. Upgrade to Django 4.2 LTS or 5.x.

### 8. No `__str__` test

`was_published_recently()` is tested thoroughly, but `Question.__str__` and `Choice.__str__` have no tests — minor, but worth noting.

---

## Points of Interest

- **Class-based views share a queryset override pattern**: both `DetailView` and `ResultsView` define `get_queryset()` identically. This is intentional separation — they could be merged into a mixin, but at this scale the duplication is clearer.
- **`app_name` namespace** in `polls/urls.py` is the Django-recommended way to avoid URL name collisions across apps. All template `{% url %}` calls use the `polls:` prefix.
- **`was_published_recently()` admin annotations** (`boolean = True`, `admin_order_field`, `short_description`) show how to make a model method a first-class admin column without any extra admin code.
- **Inline admin** (`ChoiceInline`) is a common UX pattern for parent-child models where the child has no independent life outside the parent.
- **`create_question()` test helper** with a `days` offset (positive = future, negative = past) is a clean, reusable fixture pattern for time-sensitive tests.

---

## Possible Improvements

| Priority | Improvement | Why |
|----------|-------------|-----|
| High | Upgrade to Django 4.2+ LTS | Security; modern async support |
| High | Fix race condition with `F('votes') + 1` | Data integrity under load |
| High | Move `SECRET_KEY` to environment variable | Security baseline |
| Medium | Replace SQLite with PostgreSQL | Production-grade concurrency |
| Medium | Extract temporal queryset filter to a `PublishedManager` | DRY; single place to change logic |
| Medium | Add pagination to `IndexView` | Usability beyond 5 questions |
| Medium | Add `ResultsView` tests | Test coverage gap |
| Low | Add `unique_together` on `(question, choice_text)` | Prevent duplicate choices |
| Low | Add `updated_at` timestamp to `Choice` | Audit trail for vote counts |
| Low | User authentication + one-vote-per-user enforcement | Prevent ballot stuffing |

---

## Code Style

This project follows the default Django tutorial style. Key conventions:

- **Models**: `CamelCase` class names, `snake_case` fields, `Meta` class last, `__str__` first, business methods after.
- **Views**: Class-based views for list/detail (inherit from `generic.*`), function-based views for actions that involve side effects (like `vote()`).
- **URLs**: `path()` over `re_path()`. Named patterns always. `app_name` for namespacing.
- **Templates**: Files live under `templates/<app_name>/` to avoid name collisions. Use `{% url 'ns:name' arg %}`, never hardcoded paths.
- **Static files**: Mirror the template namespace — `static/<app_name>/` — for the same collision-avoidance reason.
- **Tests**: One `TestCase` class per concern. Use `self.client` for view tests. Prefer helper factories over `setUp` fixtures for clarity.
- **Admin**: Register using `@admin.register(Model)` decorator style (the codebase uses `admin.site.register()` — both are valid, decorator style is more modern).

---

## How to Extend This

### Add a new field to Question

1. Add the field to `polls/models.py`.
2. Run `python manage.py makemigrations polls` then `python manage.py migrate`.
3. Expose it in the admin via `QuestionAdmin.fieldsets` or `list_display`.

### Add a new app (e.g. `users`)

```bash
python manage.py startapp users
```

1. Add `'users.apps.UsersConfig'` to `INSTALLED_APPS` in `settings.py`.
2. Create `users/urls.py` and include it in `mysite/urls.py`.
3. Run migrations after defining models.

### Add a new view to `polls`

1. Write the view in `polls/views.py` (class-based or function-based).
2. Add a `path(...)` entry in `polls/urls.py` with a unique `name=`.
3. Create the template at `polls/templates/polls/<view_name>.html`.
4. Write a corresponding test in `polls/tests.py`.

### Add user-based voting (one vote per user)

1. Create a `Vote` model: `ForeignKey(User)`, `ForeignKey(Choice)`, `unique_together = (('user', 'choice'),)`.
2. Wrap `vote()` with `@login_required`.
3. Check for an existing `Vote` before incrementing — replace the raw `votes` field with a count query or keep the denormalised counter and use `get_or_create`.

### Add an API endpoint

Install `djangorestframework`, add it to `INSTALLED_APPS`, create `polls/serializers.py` and `polls/api_views.py`, then mount them under `/api/polls/` in `mysite/urls.py`.

### Production checklist

- [ ] Set `SECRET_KEY` from an environment variable (`os.environ['SECRET_KEY']`)
- [ ] Set `DEBUG = False`
- [ ] Populate `ALLOWED_HOSTS`
- [ ] Switch `DATABASES` to PostgreSQL
- [ ] Configure `STATIC_ROOT` and run `collectstatic`
- [ ] Serve via gunicorn behind nginx (not `runserver`)
- [ ] Pin all dependencies in `requirements.txt` with exact versions

---

## Running the Project

```bash
python manage.py migrate          # Apply migrations
python manage.py createsuperuser  # Create admin user
python manage.py runserver        # Start dev server at http://127.0.0.1:8000
python manage.py test polls       # Run test suite
```
