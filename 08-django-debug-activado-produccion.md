# Intigriti Report — Django DEBUG=True in Production

**Program:** [UNIVERSITY] VDP
**Assets:** [TARGET-1] ([IP]), [TARGET-2] ([IP])
**VRT:** Server Security Misconfiguration > Debug Mode Enabled
**Severity:** Medium

---

## Title

Django DEBUG=True in production exposes full server configuration, stack traces, and admin URL patterns on two instances of an in-house framework ([TARGET-1], [TARGET-2])

---

## Description

Two Django applications built on the same in-house framework (`[FRAMEWORK]`) are running with `DEBUG=True` in their production settings:

1. **[TARGET-1]** ([IP]) — a website of a digital-humanities research project
2. **[TARGET-2]** ([IP]) — a website of a second research project built on the same framework

This causes Django to render detailed debug error pages for every unhandled exception (500) and every invalid URL (404). These debug pages disclose:

- Full Django settings (database engine, installed apps, cookie security flags, CORS configuration)
- Server filesystem paths
- Framework and runtime versions
- Full stack traces with source code context on 500 errors
- Complete URL routing table (100+ patterns, including all admin endpoints)

---

## Steps to Reproduce

### PoC 1 — Debug 404 with URL pattern disclosure

1. Open a browser and navigate to:
   ```
   https://[TARGET-1]/etc/passwd
   ```
2. The page displays a Django debug 404 page titled **"Page not found (404)"**.
3. The page states: *"You're seeing this error because you have `DEBUG = True` in your Django settings file."*
4. The page lists the full URL routing configuration from `django_[FRAMEWORK].urls`, including:
   - `admin/`
   - `[FRAMEWORK]/` (all admin subpaths)
   - `select2/`
   - `^ckeditor/`
   - `schemas/<str:schema_name>`
   - `p/<str:[FRAMEWORK]_page>/`
   - `^media/(?P<path>.*)$`

### PoC 2 — Debug 500 with stack trace and Django settings

1. Navigate to:
   ```
   https://[TARGET-1]/schemas/test
   ```
2. The page displays a Django debug 500 page with a `FileNotFoundError` stack trace.
3. The stack trace exposes:
   - Source file path: `/var/www/[APP]/[PROJECT]/[FRAMEWORK]/views.py`, line 101
   - Python virtualenv path: `/var/www/[APP]/[PROJECT]/venv/bin/python3`
   - Settings module: `django_[FRAMEWORK].settings`
4. Scrolling to the **Settings** section at the bottom of the error page reveals the full Django configuration, including:

   | Setting | Value |
   |---------|-------|
   | `DEBUG` | `True` |
   | `ALLOWED_HOSTS` | `['*']` |
   | `DATABASES` | `{'default': {'ENGINE': 'django.db.backends.sqlite3', ...}}` |
   | `CORS_ALLOW_ALL_ORIGINS` | `True` |
   | `CORS_ALLOW_CREDENTIALS` | `True` |
   | `SESSION_COOKIE_SECURE` | `False` |
   | `CSRF_COOKIE_SECURE` | `False` |
   | `MEDIA_ROOT` | `/var/www/[APP]/[PROJECT]/media/` |
   | `STATIC_ROOT` | `/var/www/[APP]/[PROJECT]/static` |
   | `SERVER_SOFTWARE` | `gunicorn/22.0.0` |

### PoC 3 — Admin URL pattern enumeration

1. Navigate to:
   ```
   https://[TARGET-1]/[FRAMEWORK]/nonexistent
   ```
2. The debug 404 page lists the full `[FRAMEWORK]` URL configuration, exposing 90+ admin routes:
   - `[FRAMEWORK]/configure/settings/`
   - `[FRAMEWORK]/configure/pages/create/new/`
   - `[FRAMEWORK]/configure/pages/edit/<int:pk>/`
   - `[FRAMEWORK]/configure/pages/delete/<int:pk>/`
   - `[FRAMEWORK]/configure/uploads/`
   - `[FRAMEWORK]/configure/messages/export/`
   - `[FRAMEWORK]/configure/statistics/`
   - `[FRAMEWORK]/bulk-download/json/<search_config>/`
   - `[FRAMEWORK]/bulk-download/csv/<search_config>/`
   - `[FRAMEWORK]/configure/settings/set/readonly/`
   - `[FRAMEWORK]/configure/settings/set/editable/`
   - ... and 80+ more

### Verification

On both hosts, the framework admin panel (`/[FRAMEWORK]/login/`) and the Django admin panel (`/admin/login/`) are accessible and functional. Admin functionality is behind authentication (302 redirects to login), but the debug pages are served to unauthenticated users.

---

## Second affected host — [TARGET-2]

The same vulnerability exists on the second instance. Same steps, same results:

### PoC 4 — Second host debug 500

1. Navigate to:
   ```
   https://[TARGET-2]/schemas/test
   ```
2. Django debug 500 page with `FileNotFoundError` stack trace.
3. Exposed paths: `/var/www/[APP-2]/[PROJECT-2]/`, database at `/var/www/[APP-2]/[PROJECT-2]/db.sqlite3`.
4. Source code visible: `[FRAMEWORK]/views.py`, function `display_schema_or_404`.

### PoC 5 — Second host debug 404

1. Navigate to:
   ```
   https://[TARGET-2]/search/
   ```
2. Debug 404 page with `DEBUG = True` confirmation and full URL patterns.

---

## Impact

### Confirmed

- **Full server configuration disclosure**: An unauthenticated attacker obtains the complete Django settings, revealing the database engine (SQLite3), filesystem paths, installed applications, cookie security configuration, and CORS policy.
- **Internal path disclosure**: Server filesystem paths (`/var/www/[APP]/[PROJECT]/`) and Python environment paths allow an attacker to map the server's directory structure.
- **Source code exposure**: Stack traces include source code snippets from the application, revealing internal logic.
- **Admin endpoint enumeration**: The complete URL routing table (100+ endpoints) gives an attacker a full map of the application's admin interface, including endpoints for settings modification, file uploads, bulk data downloads, and data export — all of which become known targets.
- **Framework fingerprinting**: Exact versions of Django (5.0.4), Python (3.10.12), and gunicorn (22.0.0) allow targeted CVE lookup.

### Potential (not verified)

- The combination of disclosed admin paths with known Django vulnerabilities in version 5.0.4 could facilitate further exploitation if any admin endpoints have authorization flaws beyond the login requirement.

---

## Remediation

Set `DEBUG = False` in the Django production settings file (`django_[FRAMEWORK]/settings.py` or the production-specific settings module). This is a one-line change that immediately stops Django from rendering debug error pages to users.

Additionally, consider:
- Setting `ALLOWED_HOSTS` to `['[TARGET-1]']` instead of `['*']`
- Setting `SESSION_COOKIE_SECURE = True` and `CSRF_COOKIE_SECURE = True` for HTTPS enforcement
- Configuring a custom error handler for 404 and 500 responses
