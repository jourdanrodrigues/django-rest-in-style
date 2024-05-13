# Django REST in _Style_

This is a guide to help you write Django REST applications from small to large scale.

See [project structure](./PROJECT_STRUCTURE.md).

See [abstraction layers](./ABSTRACTION_LAYERS.md).

See [testing guidelines](./TESTING.md).

## About Django Apps

A Django project should have 2 "apps" only:

- `core` containing:
  - Settings
  - Overrides of built-in management commands
  - Infrastructure tests for single server instance
  - Helpers, utilities and global overrides that are infrastructure related

- `app` containing:
  - Views
  - Models
  - Serializers
  - Locales (translations)
  - Routers definitions
  - Source of URLs in settings
  - Custom management commands
  - Helpers and utilities for application (functions/classes)

The names of the apps are a preference of mine, go with what makes sense to you.

The reason lies in the section _**Be careful about "applications"**_ from [this article][be-careful-apps-link].

## Environment

There is a pattern of splitting `settings.py` into something like `base.py`, `dev.py`, `prod.py` and `test.py`. This
tends to put the development environment away from the production environment, sometimes having large portions of code
differing between development and production.

Instead, we should have a single `settings.py` (or `settings` module) and use environment variables to set up the
application.

For certain cases, a `TEST = 'test' in sys.argv` in the settings can be used to signal that it's a test execution, but
this should be exception. Ideally, code that runs in development and tests should also run in production.

A `.env` file can be used to store these variables and [`python-dotenv`][python-dotenv-link] can be used to load them
into the application.

Should your project use Docker, and particularly `docker-compose`, the `docker-compose.yml` file can be used to set the
environment and even pair it with a `.env` file.

[python-dotenv-link]: https://github.com/theskumar/python-dotenv/
[be-careful-apps-link]: https://doordash.engineering/2017/05/15/tips-for-building-high-quality-django-apps-at-scale/
