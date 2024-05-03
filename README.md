# Django REST in _Style_

This is a guide to help you write Django REST applications consistently in a scalable manner.

## Table of Contents

- [Django Apps](#django-apps)
- [App Structure](#structure-concepts)

See the testing guidelines [here](./TESTING.md).

## Django Apps

Your Django project should have 2 "apps" only:

- `core` containing:
  - Settings
  - Overrides of built-in management commands
  - Infrastructure tests (for single server instance)
  - Helpers, utilities and global overrides for infrastructure (functions/classes)

- `app` containing:
  - Views
  - Models
  - Serializers
  - Locales (translations)
  - Routers definitions
  - Source of URLs in settings
  - Custom management commands
  - Helpers and utilities for application (functions/classes)

The "why" lies in the section _**Be careful about "applications"**_
from [this article][be-careful-apps-link].

## Structure Concepts

New concepts should be avoided as much as possible. Django and DRF structure have been time-proofed and are very flexible
to build on top of, but they require a lot of metaprogramming to come up with a dev-friendly design that works well with them.

These are the Django and DRF concepts that you should stick with:

- Authentication Classes: Responsible for authenticating users based on your authentication system.
- Permission Classes: Responsible for checking if a user has permission to perform an action.
- Views: dataset filtering, pagination and QuerySet handling (e.g. select/prefetch related); can also hold logic to switch authentication classes, permission classes and serializers.
- Serializers: mostly data validation, transformation, and two-way serialization.
- Models: database fields and object-level abstractions.
- Model Managers: database-write abstractions; avoid having multiple managers for a model, specially with overridden `get_queryset` as it's hard to remember they exist.
- QuerySets: database-read abstractions (annotations, aggregations etc.).

Some known well-justified exceptions are:

- [django-filter][django-filter-link]: A "filter" layer for views, responsible for validating and filtering QuerySets.
- [django-storages][django-storages-link]: A lib that integrates seamlessly with multiple cloud storages using Django's `Storage` concept.
- [factory-boy][factory-boy-link]: A replacement for Django's "fixtures", used to create randomized data for each test.

[be-careful-apps-link]: https://doordash.engineering/2017/05/15/tips-for-building-high-quality-django-apps-at-scale/
[django-filter-link]: https://github.com/carltongibson/django-filter/
[django-storages-link]: https://github.com/jschneier/django-storages/
[factory-boy-link]: https://github.com/FactoryBoy/factory_boy/
