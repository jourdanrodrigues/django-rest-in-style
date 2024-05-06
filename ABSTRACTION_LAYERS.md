# Abstraction Layers

New concepts should be avoided as much as possible. The structures of Django and DRF have been time-proofed and are very
flexible to build on top of, but coming up with a dev-friendly design that works well with them is not a trivial task.

These are the Django and DRF concepts that you should ideally stick with:

- Authentication Classes
- Permission Classes
- Views
- Serializers
- Models
- Model Managers
- QuerySets

Some known well-justified exceptions are:

- [django-filter][django-filter-link]: A "filter" layer for views, responsible for validating and filtering QuerySets.
- [django-storages][django-storages-link]: A lib that integrates seamlessly with multiple cloud storages using Django's
  `Storage` concept.
- [factory-boy][factory-boy-link]: A replacement for Django's "fixtures", used to create randomized data for each test.

## Authentication Classes

The authentication classes should be kept in the `authentication.py` file (or module) of your `app`. They're responsible
for finding the user based on the request. Here's an example of a custom authentication class:

```python
from rest_framework.authentication import BaseAuthentication
from rest_framework.authtoken.models import Token
from rest_framework.request import Request

from app.models import User


class CustomAuthentication(BaseAuthentication):
    def authenticate(self, request: Request) -> tuple[User | None, Token | None]:
        cookie = request.COOKIES.get('cookie')
        if not cookie:
            return None, None
        token = Token.objects.select_related('user').get(key=cookie)
        return token.user, token
```

Custom authentication classes can be attached to views by setting the `authentication_classes` attribute.

## Permission Classes

The permission classes should be kept in the `permissions.py` file (or module) of your `app`. They're responsible for
checking if the user has the necessary permissions to perform the action. Here's an example of a custom permission
class:

```python
from rest_framework.permissions import BasePermission
from rest_framework.request import Request


class CustomPermission(BasePermission):
    def has_permission(self, request: Request, view) -> bool:
        return request.user.is_authenticated

    def has_object_permission(self, request: Request, view, obj) -> bool:
        return obj.user == request.user
```

Custom permission classes can be attached to views by setting the `permission_classes` attribute.

## Views

Views should be kept in the `views.py` file (or module) of your `app`. They're responsible for handling the request and
returning a response. ViewSets should be preferred over APIViews, as they provide a better integration with the models
they related to. See the [ViewSet documentation][viewset-doc-link] for more information.

## Serializers

Serializers should be kept in the `serializers.py` file (or module) of your `app`. They're responsible for converting
complex data types, such as QuerySets and model instances, to native Python datatypes that can then be easily rendered
into JSON, XML or other content types. See the [serializer documentation][serializer-doc-link] for more information.

## Models

Models should be kept in the `models.py` file (or module) of your `app`. They're responsible for defining the structure
of the database tables. Django encourages fat models by default, but the business logic should be kept in the layer they
seem to belong.

## Model Managers

Model Managers should be kept in the `models.py` file (or module) of your `app`. Overriding the default manager should
happen when there's a need to have a custom method to write data to the database or when an annotation needs to happen
by default. An example bundled within Django is the user manager, which has a `create_user` method.

## QuerySets

QuerySets should be kept in the `models.py` file (or module) of your `app`. They're responsible for querying the
database. They should be overridden whenever there's a need to have a custom method to read data from the database.
Common reasons for this are annotations, aggregations, and filtering. Here's an example:

```python
from django.db import models


class BookQuerySet(models.QuerySet):
    def that_can_be_read_by(self: "BookQuerySet", *, user) -> "BookQuerySet":
        return self.filter(user=user, user__permissions__read=True)


class Book(models.Model):
    objects = BookQuerySet.as_manager()


# Later in code
books = Book.objects.that_can_be_read_by(user=request.user)
```

It's good to read human readability in mind when creating such methods. Avoid hiding what is done underneath, like so:

```python
class BookQuerySet(models.QuerySet):
    def that_can_be_read_by(self: "BookQuerySet", *, user) -> "BookQuerySet":
        return self.filter(
            user=user,
            user__permissions__read=True,
            active=True,  # This is not explicit and can confuse who's working with it
        ).annotate(
            read_count=models.Count('reads'),  # This is not explicit and might be annotated without need
        )
```


[serializer-doc-link]: https://www.django-rest-framework.org/api-guide/serializers/
[viewset-doc-link]: https://www.django-rest-framework.org/api-guide/viewsets/
[django-filter-link]: https://github.com/carltongibson/django-filter/
[django-storages-link]: https://github.com/jschneier/django-storages/
[factory-boy-link]: https://github.com/FactoryBoy/factory_boy/
