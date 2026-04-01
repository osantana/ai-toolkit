# Global Preferences

This file includes a set of guidelines for all projects that we will develop.

# Language

All projects are written in plain English (US variation) but must be prepared for i18n using gettext or the standard mechanism provided by the framework in use.

# General Development Instructions

## Version Control

- We use `git` for all our projects.

### Commit Messages

- Small commits with messages starting with an imperative verb (Fix, Add, Change, Rename, Refactor, Update, etc.).

## Coding Style

- Use early returns to keep the main flow of a function at the first indentation level.
- Prefer exceptions over null/flag returns. Use null only in exceptional scenarios or to maintain consistency with libraries and frameworks.
- Only handle exceptions you know how to handle. Never catch base or root exception types.
- Do not use inline documentation (e.g., docstrings, JSDoc) unless the framework requires it to document public APIs.
- Prefer clarity over brevity: use inline expressions and comprehensions only for simple cases.
- Modules must never raise exceptions on import.
- Do not shadow built-in or language-reserved names. Find a more descriptive name instead.
- Do not violate the Law of Demeter.
- Always use the latest stable version of all packages and dependencies.

## Configuration

- Follow the 12-factor app methodology for everything.
- Read all configuration and secrets from environment variables, falling back to a `.env` file at the project root.

## Code Review

- No code goes to production without passing code review.
- Keep pull requests small: deliver value while keeping changes reviewable. Split large changes into multiple PRs.
- Provide all relevant context in the PR description. Reviewers should not need to chase the linked issue for basic information.
- Deliver the best code possible on the first submission to avoid multiple review cycles.

## Testing

- Write tests using the standard testing framework for the language/project.
- Tests must be written as functions, not classes (where the framework allows).
- Place shared fixtures and helpers in a central configuration file (e.g., `tests/conftest.py`).
- Use pre-commit hooks for basic code quality checks.
- Use test coverage as a tool to find untested use cases, not as a success metric.

## Logging

- Use the language/framework logging system. Never use `print()` or `echo()` for log output.
- Emit structured, single-line JSON log records.
- Always include at minimum: log level, UTC timestamp, and a unique transaction ID.
- Never log sensitive or private data (passwords, secret keys, financial data, personal information, infrastructure details).
- Use `INFO` level as the minimum in production. Use `DEBUG` in staging and local development.
- Report unhandled exceptions to an error tracking system (e.g., Sentry). Do not mix them into transactional logs.

## Documentation

- Do not produce code documentation unless the project is FLOSS.
- All projects must have a minimal `README.md` at the root explaining how to set up the local environment, submit pull requests, and deploy the project.
- For FLOSS projects, create a `docs/` directory using the standard documentation tool for the language.

## CI/CD

- GitHub
- GitHub Actions

# REST API Guidelines

## Design

- Design the API before implementing it (design-first).
- Use a top-down approach: start from client use cases to determine what each endpoint must return. This ensures a single request satisfies a client need without over-fetching or under-fetching.

## URLs

- Version all API URLs. Use an integer prefix as the first path segment (e.g., `/v1/`, `/v2/`). Never publish an unversioned API.
- URLs represent resources (collections) or documents (individual items) — they are nouns, not verbs.
- Never put action verbs in URL paths (e.g., no `/create_card`, `/delete_user`). HTTP methods are the verbs.
- Use plural names for collections and singular names for individual documents (e.g., `/v1/orders/` and `/v1/orders/:id`).
- No trailing slash on URL paths (`/v1/orders` not `/v1/orders/`).
- Do not nest resource URLs. Each resource must have its own flat, addressable URL.
  - Bad: `/v1/printers/:printer_id/cartridges/:cartridge_id`
  - Good: `/v1/cartridges/:cartridge_id`
- Use query parameters for filtering, not URL path segments.
  - Bad: `/v1/printers/:printer_id/cartridges/`
  - Good: `/v1/cartridges/?printer_id=:printer_id`
- Negotiate content type (JSON, XML, etc.) via `Accept` and `Content-Type` headers, not URL extensions (no `.json`, `.xml` suffixes).
- Never expose sequential database primary keys in URLs. Use non-sequential IDs (e.g., UUIDs).

## HTTP Methods

- `GET`: Retrieve a document or list documents from a resource. Safe and idempotent.
- `POST`: Create a new document. The URL must identify a resource (collection), not a document.
  - Right: `POST /v1/pipes/`
  - Wrong: `POST /v1/pipes/1`
- `PUT`: Replace an existing document entirely. The URL must identify a document, not a resource. Requires sending the complete document.
  - Right: `PUT /v1/pipes/1`
  - Wrong: `PUT /v1/pipes/`
- `PATCH`: Partially update a document. Can modify any number of fields (N ≥ 1) in a single request. The body describes *how* to alter the document, not which single field to change.
- `DELETE`: Remove a document.

### Idempotency

- Duplicate `POST` requests (same resource created twice) must return `303 See Other` with a `Location` header pointing to the existing resource.
- Duplicate `PUT` and `PATCH` requests with no effective change must return `304 Not Modified`.

## State Management

- Design reactive APIs: trigger state transitions by setting meaningful fields, not by passing an explicit status value.
  - Right: `PATCH /v1/orders/1 {"approved_at": "2026-04-01T12:00:00Z"}` → order transitions to `approved`
  - Wrong: `PATCH /v1/orders/1 {"status": "approved"}`
- Do not allow clients to force arbitrary state transitions by passing a status value directly.
- Avoid boolean flag parameters. Model behavior as structured parameters instead.
  - Bad: `free_shipping=true`
  - Good: `{"shipping": {"payer": "store", "rate": 0}}`

## Status Codes

- Use the semantically correct HTTP status code for every response.
- `GET` on a resource with zero documents → `200 OK` with an empty list. Never return `404` for an empty collection.
- `404 Not Found` → only when the endpoint itself does not exist.
- `4xx` signals a client error: the client must correct the request before retrying.
- `5xx` signals a server error: the client may retry the same request later.
- Do not return `4xx` or `5xx` for business rule conflicts (e.g., a duplicate that the system intentionally rejects). Return `200 OK` with a `Content-Location` header pointing to the already-existing resource.
- Return `404 Not Found` instead of `403 Forbidden` for anonymous (unauthenticated) access. Never confirm that a resource exists to an unauthenticated caller.
- Be consistent: a given situation must always produce the same status code across all endpoints.

## Pagination

- Paginate all list endpoints.
- Do not use page-number-based pagination (e.g., `?page=2`). Page numbers are unstable under concurrent writes: items can shift between pages while a client is iterating, causing duplicates or skipped records.
- Prefer cursor-based pagination: the server returns an opaque `cursor` (or `next`/`previous` token) that the client passes on the next request. This is stable under concurrent writes and scales better with large datasets.
- Use `limit` and `offset` only when cursor-based pagination is impractical (e.g., the client needs random access to arbitrary positions).

# Python Development Instructions

## Python Coding Style

See the `ruff` configuration in the `pyproject.toml` sample below for enforced style settings.

- PEP 8 with 120-character lines (enforced by ruff).
- Use type annotations only where required by the project or framework. Do not annotate everything.
- Avoid `elif` and `else` wherever possible.
- Use `None` as a flag return only in exceptional scenarios or to maintain consistency with libraries and frameworks.
- Never catch `Exception` or `BaseException` — only catch specific exception types you know how to handle.
- Do not use docstrings unless the framework requires them to document public APIs.
- Avoid `lambda` except for short expressions used as a `key` argument.
- Do not use `dict()` to construct dictionaries. Prefer classes or dataclasses.
- Properties and `__init__()` must never raise exceptions.
- Do not use name mangling (`__double_underscore`).
- Never modify `sys.path`.
- Never use `mock.patch` in tests.

## Web Applications

See [Python Configuration Files](#python-configuration-files) at the end of this document for sample files.

### Backend

- Django
- PostgreSQL

#### Common Libraries

- `uvicorn`: WSGI/ASGI application server
- `whitenoise`: serve static files
- `psycopg`: PostgreSQL driver
- `prettyconf`: environment variable config (12-factor apps)
- `dj-database-url`: used with `prettyconf` to parse `DATABASE_URL`
- `django-storages`: static and media file storage
- `django-allauth[socialaccount]`: signup/signin

### Frontend

- HTMX
- Tabler.io
- Django Template Language

### Development Tools

- `make`
- `uv`
- `pre-commit`
- `ruff`
- `mypy`
- `django-stubs`
- `pytest`
- `pytest-cov`
- `pytest-asyncio`
- `pytest-django`

## Configuration Rules

- Read configuration and secrets from environment variables using `prettyconf`, falling back to a `.env` file at the project root.
- Serve all static files through `whitenoise`.
- Use pytest for testing, with shared fixtures in `tests/conftest.py`.
- For FLOSS projects, use `sphinx` for documentation (add a documentation dependency group in `pyproject.toml`).

### Commands

Core development commands are implemented as Makefile targets:

- `make test`: Run all tests with coverage
- `make lint`: Run code linting with ruff
- `make format`: Format the code with ruff
- `make typecheck`: Run type checking with mypy

### Deployment

#### Serverless

- Terraform for IaC
- AWS Lambda (or Google Cloud Run)
- Aurora (or AlloyDB) for PostgreSQL
- EventBridge (or Pub/Sub)
- API Gateway
- Cloudflare (for SSL, DNS, CDN, and WAF)

#### PaaS

- Heroku Dyno
- Heroku PostgreSQL
- EventBridge or Pub/Sub
- The application sits behind Cloudflare, which manages DNS and SSL via a wildcard certificate (`*.[project-name].io`).
- User-uploaded media files are stored in an AWS S3 bucket named `[project-name]-media`.

### Project Structure for Django Projects

- `[root]/`
    - `apps/` — Django apps (one package per app)
    - `frontend/`
        - `assets/` — Static assets (images, fonts, stylesheets)
            - `css/`
            - `images/`
            - `js/`
            - `fonts/`
            - `vendors/`
                - `tabler/` — Pre-compiled/bundled Tabler dist assets
        - `templates/` — Django templates (organized by app)
    - `locale/{en,pt_BR}` — i18n
    - `tests/` — All tests (organized by app)
    - `[project-name]/` — Main Django project module
    - `static/` — Target directory for collected static files

### Python Configuration Files

#### `pyproject.toml`

```toml
[project]
name = "[project-name]"
version = "0.1.0"
description = "Referal links management for content creators"
readme = "README.md"
requires-python = ">=3.14"
dependencies = [
    "django>=6.0.1",
    "psycopg>=3.3.2",
    "dj-database-url>=2.3.0",
    "prettyconf>=2.3.0",
    "whitenoise>=6.11.0",
    "django-allauth[socialaccount]>=65.0.0",
    "uvicorn>=0.42.0",
]

[dependency-groups]
dev = [
    "pre-commit",
    "ruff",
    "mypy",
    "django-stubs",
    "pytest",
    "pytest-cov",
    "pytest-asyncio",
    "pytest-django",
]

[tool.ruff]
line-length = 120    # PEP 8 allows 79; 120 is the modern team compromise (enforced by formatter too)
target-version = "py314"  # Unlocks syntax modernization rules that target Python 3.14 features
exclude = [
    "migrations",  # Django auto-generated files: reformatting breaks squashing and diffing
    "manage.py",   # Django entry point: generated boilerplate, not subject to project style
]

[tool.ruff.lint]
# Preview rules are still being stabilized by the ruff team but are safe to use.
# They expose additional checks and fixes before graduating to stable defaults.
preview = true

select = [
    # ── Core correctness ─────────────────────────────────────────────────────────
    "E",    # pycodestyle errors   – PEP 8 violations: indentation, spacing, line length (E501), etc.
    "W",    # pycodestyle warnings – trailing whitespace, missing newline at EOF, invalid escape sequences (W605)
    "F",    # pyflakes             – undefined names, unused imports, redefined variables, star-import issues
    "B",    # flake8-bugbear       – likely bugs and design problems: mutable default arguments (B006),
            #                        function calls in defaults (B008), redundant assertions (B011), etc.

    # ── Imports ──────────────────────────────────────────────────────────────────
    "I",    # isort – deterministic import ordering, grouping (stdlib / third-party / local)

    # ── Naming ───────────────────────────────────────────────────────────────────
    "N",    # pep8-naming – PEP 8 naming conventions: CapWords classes, snake_case functions/vars

    # ── Python modernization ─────────────────────────────────────────────────────
    "UP",   # pyupgrade – upgrade syntax to target Python version: f-strings, union types (X | Y), etc.
    "FURB", # refurb    – additional modernization beyond pyupgrade: pathlib, removeprefix/suffix, bit_count, etc.
    "FLY",  # flynt     – prefer f-strings for str.join() patterns and %-formatting (complements UP)

    # ── Built-in shadowing ────────────────────────────────────────────────────────
    "A",    # flake8-builtins – detect variables/arguments that shadow Python built-ins (list, id, type, etc.)

    # ── Code quality and readability ──────────────────────────────────────────────
    "C4",   # flake8-comprehensions – prefer proper comprehension forms over map()/filter()/list(generator)
    "ISC",  # flake8-implicit-str-concat – catch accidental "foo" "bar" merges that look like one string
    "PIE",  # flake8-pie – miscellaneous improvements: unnecessary pass, prefer isinstance(), etc.
    "SIM",  # flake8-simplify – simplify conditionals (SIM108), context managers, and return statements
    "RSE",  # flake8-raise   – ensure raise uses instantiated exceptions: raise Foo() not raise Foo
    "RET",  # flake8-return  – consistent return patterns; remove redundant else/elif after return (RET505)

    # ── Performance ───────────────────────────────────────────────────────────────
    "PERF", # Perflint – flag performance antipatterns: list.append in loop (PERF401), manual list sum, etc.

    # ── Datetime safety ───────────────────────────────────────────────────────────
    "DTZ",  # flake8-datetimez – require timezone-aware datetimes throughout; with USE_TZ=True this prevents
            #                     subtle bugs where naive and aware datetimes silently mix (DTZ001, DTZ011, etc.)

    # ── Miscellaneous anti-patterns ───────────────────────────────────────────────
    "PGH",  # pygrep-hooks – bare `# type: ignore` without error codes (PGH003), deprecated mock aliases (PGH004)

    # ── Debug / dead code ─────────────────────────────────────────────────────────
    "T20",  # flake8-print – flag leftover print() calls; use Django's logging framework instead
    "ERA",  # eradicate    – detect commented-out dead code that should be removed or restored

    # ── Exception handling ────────────────────────────────────────────────────────
    "BLE",  # flake8-blind-except – disallow bare `except Exception` without re-raising or specific handling

    # ── Testing ───────────────────────────────────────────────────────────────────
    "PT",   # flake8-pytest-style – consistent pytest idioms: raises context manager, fixture scopes, marks

    # ── Arguments ─────────────────────────────────────────────────────────────────
    "ARG",  # flake8-unused-arguments – catch parameters that are defined but never read inside the function

    # ── Pylint ────────────────────────────────────────────────────────────────────
    "PL",   # pylint – ported rules: magic numbers (PLR2004), comparison anti-patterns, cyclomatic complexity

    # ── Ruff-native ───────────────────────────────────────────────────────────────
    "RUF",  # ruff-specific rules – implicit optional (RUF013), mutable class defaults (RUF012), ambiguous names
]

ignore = [
    # Lambda assignments (e.g. key = lambda x: x.name) are idiomatic in Python for
    # short sort/filter keys; wrapping them in def would reduce readability at call sites.
    "E731",

    # Django views, model methods, management commands, and admin classes regularly
    # exceed 5 positional arguments by design (request + many kwargs). Enforcing this
    # limit would require artificial refactors that hurt clarity without helping quality.
    "PLR0913",  # too-many-arguments
    "PLR0917",  # too-many-positional-arguments
]

[tool.ruff.analyze]
detect-string-imports = true  # Resolve string-based imports for import-cycle analysis

[tool.ruff.format]
quote-style = "single"  # Consistent with the project's existing single-quote convention

[tool.ruff.lint.per-file-ignores]
"tests/**" = [
    # Pytest fixtures commonly accept many parameters to compose test scenarios
    "PLR0917",  # too-many-positional-arguments
    "PLR0913",  # too-many-arguments

    # Fixtures are often declared as function parameters even when used only for
    # their side-effects (e.g. db setup), not for their return value
    "ARG001",   # unused-function-argument

    # Literal comparison values in assertions are intentional and self-documenting
    "PLR2004",  # magic-value-comparison

    # Timezone-naive datetimes are acceptable in test data factories and fixtures
    # where timezone correctness is not what is being tested
    "DTZ001",   # datetime.datetime.now() without tz
    "DTZ003",   # datetime.datetime.utcnow() usage
    "DTZ011",   # datetime.date.today() usage
]

[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "[project-name].settings"
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
testpaths = ["tests"]

[tool.mypy]
python_version = "3.14"
plugins = ["mypy_django_plugin.main"]
exclude = [".venv"]
ignore_missing_imports = true

[tool.django-stubs]
django_settings_module = "[project-name].settings"
```

#### `Makefile`

```Makefile

SHELL := /bin/bash

UV := uv
RUFF = $(UV) run ruff
MYPY = $(UV) run mypy
PYTEST = $(UV) run pytest
MANAGE = $(UV) run manage.py
SRC=[project-name] apps
TEST=tests

# Targets
.make/ensure-uv:
	@$(UV) --version > /dev/null 2>&1 || ./bin/install-uv.sh
	@mkdir -p .make/ && touch .make/ensure-uv

uv.lock: .make/ensure-uv pyproject.toml
	$(UV) lock --no-upgrade
	touch uv.lock  # force update even when not changed

.make/setup: uv.lock
	$(UV) sync --locked; \
	$(UV) run pre-commit install;
	@mkdir -p .make/ && touch .make/setup

.env: local.env
	@cp local.env .env || echo "NOTE: review your .env file comparing with local.env"
	@touch .env

.make/init: uv.lock .make/setup
	./bin/createdb.sh .env;
	$(MANAGE) migrate
	@mkdir -p .make/ && touch .make/init


# Development
setup: .make/setup

init: .make/init

update-deps: .make/setup
	$(UV) lock -U
	$(UV) sync -U

lint: .make/setup
	$(RUFF) check $(SRC) $(TEST)
	$(RUFF) format --check $(SRC) $(TEST)

format: .make/setup
	$(RUFF) format $(SRC) $(TEST)
	$(RUFF) check $(SRC) $(TEST) --fix

type-check: .make/setup
	$(MYPY) $(SRC)

test: .make/setup
	$(PYTEST) -v $(TEST) $(ARGS)

coverage: .make/init
	$(PYTEST) -v --cov=. --cov-report=term --cov-report=xml --cov-report=term-missing $(TEST) $(ARGS)

clean:
	rm -f .make/[a-z]*

clean-all: clean
	rm -f .env

# Django Commands
runserver: .make/init
	$(MANAGE) $@ 0.0.0.0:8000 $(ARGS)

migrations: .make/init
	$(MANAGE) makemigrations $(ARGS)

migrate: .make/init
	$(MANAGE) $@ $(ARGS)

collectstatic: .make/init
	$(MANAGE) $@ --no-input $(ARGS)

dbshell: .make/init
	$(MANAGE) $@ $(ARGS)

messages: .make/init
	$(MANAGE) makemessages -a -s $(ARGS)

compilemessages: .make/init
	$(MANAGE) $@ $(ARGS)

help:
	@echo "Available targets:"
	@echo ""
	@echo "  Development:"
	@echo "    setup:             install all requirements and prepare the environment for local development"
	@echo "    init:              initialize the local configuration and database"
	@echo "    update-deps:       update all dependencies"
	@echo "    lint:              run linters on all files"
	@echo "    format:            run formatters on all files"
	@echo "    type-check:        run type checkers on the project"
	@echo "    test:              run all tests (use ARGS=\"...\" to pass extra arguments to pytest)"
	@echo "    coverage:          run all tests with coverage report"
	@echo "    clean:             remove all makefile targets"
	@echo "    clean-all:         remove all makefile targets and .env configuration file"
	@echo ""
	@echo "  Project Specific (use ARGS=\"...\" to pass extra arguments to management commands):"
	@echo "    runserver:         run local server"
	@echo "    makemigrations:    create a new database migration"
	@echo "    migrate:           run database migrations"
	@echo "    collectstatic:     collect static files into STATIC_ROOT directory"
	@echo "    sqlshell:          connect to the postgres database configured at DATABASE_URL in .env file"

.PHONY: setup init update-deps lint format type-check test coverage clean clean-all runserver makemigrations migrate collectstatic sqlshell help
```

#### `.pre-commit-config.yaml`

```yaml
# pre-commit configuration file
#
# WARNING:
# We use pre-commit exclusively for checking/linting purpose and
# not to format files. If you want to format your files
# run `make format`.

repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: "v6.0.0"
    hooks:
      - id: check-yaml
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-ast
      - id: check-json
      - id: check-merge-conflict
      - id: debug-statements
      - id: check-symlinks
      - id: check-shebang-scripts-are-executable
      - id: check-toml
      - id: detect-aws-credentials
      - id: detect-private-key
      - id: forbid-submodules
  - repo: https://github.com/charliermarsh/ruff-pre-commit
    rev: "v0.14.14"
    hooks:
      - id: ruff-check
        args: [ --config=pyproject.toml ]
  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: "0.9.28"
    hooks:
      - id: uv-lock
```

#### `.gitignore`

```gitignore
# OS
.DS_Store

# Python
__pycache__/
*.py[cod]
*.so
.venv/

# Django
db.sqlite3
db.sqlite3-journal
/media/
/static/

# IDE
.idea/
.vscode/
.junie/
*.swp
*.swo

# Tools
.pytest_cache/
.mypy_cache/
.coverage
coverage.xml
htmlcov/
.cache/

# Project
.make/
.env
```


#### `settings.py`

Standard settings file for a Django project.

```python
from pathlib import Path

from dj_database_url import parse as db_url_parser
from django.utils.translation import gettext_lazy as _
from prettyconf import config

BASE_DIR = Path(__file__).resolve().parent.parent
DEBUG = config('DEBUG', default=False, cast=config.boolean)

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.sites',
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
    'allauth.socialaccount.providers.facebook',
    'apps.accounts',
    'apps.links',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'allauth.account.middleware.AccountMiddleware',
]

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'frontend' / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                '[project-name].context_processors.assets',
            ],
        },
    },
]

# Database
DATABASES = {
    'default': config('DATABASE_URL', cast=db_url_parser),
}
DATABASES['default']['CONN_MAX_AGE'] = None  # always connected
DATABASES['default']['ATOMIC_REQUESTS'] = True

# Security
SECRET_KEY = config('SECRET_KEY')
ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='*', cast=config.list)
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

PASSWORD_HASHERS = [
    # Set PASSWORD_HASHER=MD5PasswordHasher to make test running faster
    'django.contrib.auth.hashers.' + config('PASSWORD_HASHER', default='PBKDF2PasswordHasher'),
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
    'django.contrib.auth.hashers.ScryptPasswordHasher',
]

# Authentication
AUTH_USER_MODEL = 'accounts.User'
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]
SITE_ID = 1
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'

# Allauth settings
ACCOUNT_USER_MODEL_USERNAME_FIELD = None
ACCOUNT_LOGIN_METHODS = {'email'}
ACCOUNT_SIGNUP_FIELDS = ['email*', 'password1*', 'password2*']
ACCOUNT_UNIQUE_EMAIL = True
ACCOUNT_EMAIL_VERIFICATION = config('ACCOUNT_EMAIL_VERIFICATION', default='optional')
ACCOUNT_LOGIN_ON_EMAIL_CONFIRMATION = True
ACCOUNT_LOGOUT_ON_GET = False
ACCOUNT_ADAPTER = 'apps.accounts.adapters.AccountAdapter'
SOCIALACCOUNT_ADAPTER = 'apps.accounts.adapters.SocialAccountAdapter'
SOCIALACCOUNT_AUTO_SIGNUP = True
SOCIALACCOUNT_EMAIL_AUTHENTICATION = True
SOCIALACCOUNT_EMAIL_AUTHENTICATION_AUTO_CONNECT = True

# Social providers configuration
SOCIALACCOUNT_PROVIDERS = {
    'google': {
        'APP': {
            'client_id': config('GOOGLE_CLIENT_ID', default=''),
            'secret': config('GOOGLE_CLIENT_SECRET', default=''),
        },
        'SCOPE': ['profile', 'email'],
        'AUTH_PARAMS': {'access_type': 'online'},
    },
    'facebook': {
        'APP': {
            'client_id': config('FACEBOOK_APP_ID', default=''),
            'secret': config('FACEBOOK_APP_SECRET', default=''),
        },
        'METHOD': 'oauth2',
        'SCOPE': ['email', 'public_profile'],
        'FIELDS': ['id', 'email', 'name', 'first_name', 'last_name', 'picture'],
    },
}

# Internationalization
TIME_ZONE = 'UTC'
USE_I18N = True
USE_L10N = True
USE_TZ = True
LANGUAGE_CODE = 'en-us'
LANGUAGES = (
    ('en', _('English')),
    ('pt-br', _('Brazilian Portuguese')),
)
LOCALE_PATHS = (str(BASE_DIR / 'locale'),)

# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'static'
STATICFILES_DIRS = [
    BASE_DIR / 'frontend' / 'assets',
]

# Logging
LOG_LEVEL = config('LOG_LEVEL', default='WARNING')
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'null': {
            'class': 'logging.NullHandler',
        },
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'loggers': {
        'root': {
            'handlers': ['console'],
            'level': LOG_LEVEL,
        },
        'django': {
            'handlers': ['console'],
            'level': LOG_LEVEL,
        },
    },
}

WSGI_APPLICATION = '[project-name].wsgi.application'
ROOT_URLCONF = '[project-name].urls'
```

#### `local.env`

```shell
DEBUG=True
LOG_LEVEL=INFO
PASSWORD_HASHER=MD5PasswordHasher
DATABASE_URL=postgresql://[project-name]:[project-name]@localhost/[project-name]
SECRET_KEY="sekure-key"
```

#### `.github/workflows/ci.yaml`

```yaml
name: Code Quality and Test Checks (Staging)

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  contents: read
  checks: write
  pull-requests: write
  id-token: write

jobs:
  static:
    name: Code Quality Check
    runs-on: ubuntu-latest
    environment: development

    env:
      DEBUG: True
      LOG_LEVEL: INFO
      PASSWORD_HASHER: MD5PasswordHasher
      DATABASE_URL: postgresql://[project-name]:[project-name]@localhost/[project-name]
      SECRET_KEY: "secret-key-for-testing"

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"

      - name: Setup environment
        run: |
          make setup

      - name: Run Linter
        run: |
          make lint

      - name: Run Type Check
        run: |
          make type-check

  tests:
    name: Tests
    runs-on: ubuntu-latest
    environment: development

    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: [project-name]
          POSTGRES_PASSWORD: [project-name]
          POSTGRES_DB: [project-name]
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      DEBUG: True
      LOG_LEVEL: INFO
      PASSWORD_HASHER: MD5PasswordHasher
      DATABASE_URL: postgresql://[project-name]:[project-name]@localhost:5432/[project-name]
      SECRET_KEY: "secret-key-for-testing"

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"

      - name: Setup environment
        run: |
          make setup

      - name: Run Tests
        run: |
          make coverage
```

## Writings

Projects for books.

### Dependencies

- Python
- Asciidoc
