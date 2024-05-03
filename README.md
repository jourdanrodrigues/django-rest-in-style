# Django REST in _Style_

This is a guide to help you write Django REST applications from small to large scale.

## Table of Contents

- [Project Structure](#project-structure)
- [Django Apps](#django-apps)
- [Abstraction Layers](#abstraction-layers)

See the testing guidelines [here](./TESTING.md).

## Project Structure

The project structure should roughly look like this:

    project_dir/
    ├── app/
    │    ├── __init__.py
    │    ├── migrations/
    │    ├── tests/
    │    ├── admin.py
    │    ├── apps.py
    │    ├── authentication.py
    │    ├── filters.py
    │    ├── models.py
    │    ├── permissions.py
    │    ├── serializers.py
    │    ├── urls.py
    │    └── views.py
    ├── core/
    │   ├── __init__.py
    │   ├── tests/
    │   ├── asgi.py
    │   ├── settings.py
    │   ├── storage.py
    │   └── wsgi.py
    ├── manage.py
    ├── poetry.lock
    └── pyproject.toml

Each of the Python files can be turned into a Python module as the codebase grows.

    project_dir/
    └── app/
        └── models/
            ├── __init__.py
            ├── user.py
            └── profile.py


Considering the design decision of only importing from the top level of these modules, the files can be prepended with
an underscore (`_`) and have what should be "exposed" imported in the `__init__.py`. IDEs don't work well with this
approach with more than 1 level of nesting.

    project_dir/
    └── app/
        ├── __init__.py
        └── models/
            ├── __init__.py
            ├── _user.py
            └── _profile.py

    # _user.py
    class User:
        pass

    # _profile.py
    class Profile:
        pass

    # __init__.py
    from app.models._user import User
    from app.models._profile import Profile

## Django Apps

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

The reason lies in the section _**Be careful about "applications"**_ from [this article][be-careful-apps-link].

## Abstraction Layers

New concepts should be avoided as much as possible. Django + DRF structure have been time-proofed and is very flexible
to build on top of, but coming up with a dev-friendly design that works well with it is not a trivial task.

These are the Django and DRF concepts that you should stick with:

- Authentication Classes: Responsible for authenticating users based on your authentication system.
- Permission Classes: Responsible for checking if a user has permission to perform an action.
- Views: dataset filtering, pagination and QuerySet handling (e.g. select/prefetch related); can also hold logic to
switch authentication classes, permission classes and serializers.
- Serializers: mostly data validation, transformation, and two-way serialization.
- Models: database fields and object-level abstractions.
- Model Managers: database-write abstractions; avoid having multiple managers for a model, specially with overridden
`get_queryset` as it's hard to remember they exist.
- QuerySets: database-read abstractions (annotations, aggregations etc.).

Some known well-justified exceptions are:

- [django-filter][django-filter-link]: A "filter" layer for views, responsible for validating and filtering QuerySets.
- [django-storages][django-storages-link]: A lib that integrates seamlessly with multiple cloud storages using Django's
`Storage` concept.
- [factory-boy][factory-boy-link]: A replacement for Django's "fixtures", used to create randomized data for each test.

## Environment

There is a pattern of separating `settings.py` into something like `base.py`, `dev.py`, `prod.py` and `test.py`. This
tends to put the development environment away from the production environment, sometimes having large portions of the
codebase differing between development and production.

Instead, we should have a single `settings.py` (or `settings` module) and use environment variables to set up the
application.

A `.env` file can be used to store these variables and [`python-dotenv`][python-dotenv-link] can be used to load them
into the application.

Should your project use Docker, and particularly `docker-compose`, the `docker-compose.yml` file can be used to set the
environment and even pair it with a `.env` file.

[python-dotenv-link]: https://github.com/theskumar/python-dotenv/
[be-careful-apps-link]: https://doordash.engineering/2017/05/15/tips-for-building-high-quality-django-apps-at-scale/
[django-filter-link]: https://github.com/carltongibson/django-filter/
[django-storages-link]: https://github.com/jschneier/django-storages/
[factory-boy-link]: https://github.com/FactoryBoy/factory_boy/
