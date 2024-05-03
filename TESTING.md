# Testing Guidelines

## Table of Contents

- [Test names](#test-names)
- [Test assertions](#test-assertions)
- [Test structure: Arrange-Act-Assert](#test-structure-arrange-act-assert)
- [Unit Tests](#unit-tests)
  - [Testing functions and classes](#testing-functions-and-classes)
- [Integration](#integration)
  - [Web API](#web-api)

## Test names

Test names should follow the "proposition" pattern: `test_when_["and" conditions]_then_[expected result]`.

As a side effect, propositions should enforce the pattern of only one assertion per test.

Each test should assert "and" conditions only. The "or" conditions should always be asserted in a different test.

## Test assertions

Tests should only have one assertion.

If the test is asserting a single concept that requires multiple assertions, then go for it but having multiple
assertions potentially requires multiple runs to fully fix a broken test.

Bundling the assertions in a single `assertListEqual`, `assertDictEqual` etc. call is an acceptable way to break this rule.

Assertion abstractions should be based on this. Here's an example of a very common assertion abstraction:

```python
import json

from rest_framework import status
from rest_framework.renderers import JSONRenderer
from rest_framework.response import Response
from rest_framework.test import APITestCase as DRFAPITestCase


def extract_response_data(response: Response):
    """
    Extracts a diff-friendly response data from a DRF Response object.
    """
    if not hasattr(response, "data"):
        return response.content
    return json.loads(JSONRenderer().render(response.data))


class APITestCase(DRFAPITestCase):
    def assertResponse(self, response: Response, status_code: int, expected_data) -> None:
        self.assertListEqual(
            [response.status_code, extract_response_data(response)],
            [status_code, expected_data],
        )

    def assertOkResponse(self, response: Response, expected_data: list | dict | None) -> None:
        self.assertResponse(response, status.HTTP_200_OK, expected_data)

    def assertNoContentResponse(self, response: Response) -> None:
        self.assertResponse(response, status.HTTP_204_NO_CONTENT, None)

    def assertNotFoundResponse(self, response: Response, message: str | None = None) -> None:
        self.assertResponse(response, status.HTTP_404_NOT_FOUND, {"detail": message or "Not found."})

    def assertUnauthorizedResponse(self, response: Response, expected_data: dict | None = None) -> None:
        data = expected_data or {"detail": "Authentication credentials were not provided."}
        self.assertResponse(response, status.HTTP_401_UNAUTHORIZED, data)

    # Other response assertions...
```

## Test structure: Arrange-Act-Assert

- [Automation Panda: Arrange-Act-Assert][arrange-act-assert-panda]
- [C2: Arrange-Act-Assert][arrange-act-assert-c2]

Writing test code is not the same as writing production code.

Production code should be easy to make changes that have a wide effect on the codebase, where concepts like DRY (Don't Repeat Yourself) come in.
The goal is to get the business logic right with the least amount of effort.

Test code, on the other hand, should be easy to understand and easy to be fixed with the least amount of effort.
This means that it's better to have explicit duplication in the test code (completely breaking DRY) than to have to navigate to another method or file to understand the test.

This is an example of some test cases with reusability in mind:

```python
# A test file for an endpoint
from app.tests.utils import add_book_to_user, TestAssertionsMixin, TestLoggedIn
from rest_framework import status


class TestGet(TestLoggedIn, TestAssertionsMixin):
    def setUp(self):
        # Not clear what is the setup
        super().setUp()
        self.book = add_book_to_user(self.user)

    def test_when_user_is_not_logged_in_then_returns_unauthorized(self):
        self.logout_user()  # Was there a user logged in? From where?

        response = self.client.get(f'/api/users/{self.user.id}/books/')

        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_when_user_logged_in_has_books_returns_its_books(self):
        # Is not clear what is the setup
        response = self.client.get(f'/api/users/{self.user.id}/books/')

        # Is not clear what is being asserted
        self.assertBookIsInResponse(response, self.book)
```

```python
# Test utilities at "project_dir/app/tests/utils.py"
from rest_framework.test import APITestCase
from rest_framework import status
from app.factories import BookFactory, UserFactory, TokenFactory


def add_book_to_user(user, book=None):
    book = book or BookFactory(name='The Lord of the Rings')
    user.books.add(book)
    return book


class TestAssertionsMixin:
    def assertBookIsInResponse(self, response, book):
        if response.status_code != status.HTTP_200_OK:
            raise AssertionError(f'Expected status code 200, got {response.status_code} and response:\n{response.content}')
        if isinstance(response.data, list):
            book_data = response.data[0]
        else:
            book_data = response.data

        book_fields = ['id', 'title', 'author', 'year']
        for field in book_fields:
            self.assertEqual(book_data[field], getattr(book, field))


class TestLoggedIn(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.token = TokenFactory(user=self.user)
        self.client.credentials(HTTP_AUTHORIZATION=f'Token {self.token.value}')

    def logout_user(self):
        self.client.credentials(HTTP_AUTHORIZATION='')
```

Now the same set test cases but with Arrange-Act-Assert structure

```python
from uuid import uuid4
from rest_framework.test import APITestCase
from rest_framework import status
from app.factories import BookFactory, UserFactory, TokenFactory


class TestGet(APITestCase):
    def test_when_user_is_not_logged_in_then_returns_unauthorized(self):
        # No arrange needed

        # Act
        response = self.client.get(f'/api/users/{uuid4()}/books/')

        # Assert
        expected_data = {'detail': 'Authentication credentials were not provided.'}
        self.assertListEqual(
            [response.status_code, response.data],
            [status.HTTP_401_UNAUTHORIZED, expected_data],
        )

    def test_when_user_logged_in_has_books_returns_its_books(self):
        # Arrange
        user = UserFactory()
        book = BookFactory()
        user.books.add(book)
        token = TokenFactory(user=user)
        self.client.credentials(HTTP_AUTHORIZATION=f'Token {token.value}')

        # Act
        response = self.client.get(f'/api/users/{book.id}/books/')

        # Assert
        book_data = {
            'id': book.id,
            'title': book.title,
            'author': book.author,
            'year': book.year,
        }
        self.assertListEqual(
            [response.status_code, response.data],
            [status.HTTP_200_OK, [book_data]],
        )
```

This is not to say abstractions shouldn't be created for testing purposes. Testing abstractions should not hide away the test's intent.
Having excessive logic in the test utilities will make the tests harder to understand, and therefore, harder to be fixed.

Django also has a `setUpTestData` method that can be used to create data that is shared among all tests in the test case class.
That should be used only when the data is the exact same for all tests in the class.
Having to rework data for a test means writing logic, which makes the test harder to understand.

## Unit Tests

Normally, unit tests inherit from [`django.test.SimpleTestCase`][simpletestcase-django-doc] as this class doesn't set up a test database.

The folder structure for the tests below should be as follows:

```
project_dir/
├── app/
│   ├── utils.py
│   ├── serializers.py
│   ├── tests/
│   │   ├── utils/
│   │   │   ├── test_split_names.py
│   │   │   ├── test_parser.py
│   │   ├── serializers/
│   │   │   ├── test_address_serializer.py
```

### Testing functions and classes

You should always test the class' methods separately. The test class name should be named `Test<Method>`.

You should not take into account any underscores (`_`) in the names (e.g. testing `__init__` would have `TestInit`).

You should test any class, class method or function beginning with an underscore (`_`).
They are considered private and should be test only indirectly.

```python
# File at "project_dir/app/utils.py"
import json


def split_names(full_name: str) -> list[str]:
    return full_name.split(' ')


class Parser:
    @staticmethod
    def to_json(json_string: str) -> dict:
        return json.loads(json_string)

# File at "project_dir/app/serializers.py"
from rest_framework import serializers


class AddressSerializer(serializers.Serializer):
    street = serializers.CharField()
```

```python
# Test at "project_dir/app/tests/utils/test_split_names.py"
from django.test import SimpleTestCase

from app.utils import split_names


class TestFunction(SimpleTestCase):
    def test_when_string_without_spaces_is_sent_returns_list_with_one_element(self):
        value = split_names('first last')

        self.assertListEqual(value, ['first', 'last'])

# Test at "project_dir/app/tests/serializers/test_address_serializer.py"
from django.test import SimpleTestCase

from app.serializers import AddressSerializer


class TestInit(SimpleTestCase):
    def test_when_instance_is_created_right_properties_are_applied(self):
        data = {'street': 'Test Aloha'}

        # Serializes are an exception because they have a bulky API, so these 3 lines are part of the "act"
        serializer = AddressSerializer(data=data)
        serializer.is_valid(raise_exception=True)
        instance = serializer.save()

        self.assertEqual(instance.street, data['street'])

# Test at "project_dir/app/tests/utils/test_parser.py"
from django.test import SimpleTestCase

from app.utils import Parser


class TestToJson(SimpleTestCase):
    def test_when_json_string_is_valid_returns_dictionary(self):
        value = Parser().to_json('{"key": "value"}')

        self.assertDictEqual(value, {'key': 'value'})
```

## Integration

These can inherit either from "[rest_framework.test.APITestCase][apitestcase-drf-doc]" (for API calls) or
"[django.test.TestCase][testcase-django-doc]"

### Web API

In terms of folder structure, I recommend putting endpoint tests under an `endpoints` folder and have the folder
structure mimic the endpoint path, with a `test_resource.py` file to hold the tests for that endpoint.

Naming here is totally flexible, go with what works best for you.

In order to maintain the API contract, it's recommended to have at one test checking for the entire payload per endpoint.
This is to ensure that the API is not returning:
- more data than expected;
- data in a different format;
- data in a different order;
- with missing data.

That way, any payload changes will be caught by the tests and will have to be intentionally changed.

The whole test structure for the examples above should be:

    project_dir/
    ├── app/
    │   ├── factories.py
    │   ├── models.py
    │   ├── serializers.py
    │   ├── tests/
    │   │   ├── endpoints/
    │   │   │   ├── api/
    │   │   │   │   ├── books/
    │   │   │   │   │   ├── test_resource.py
    │   │   │   │   │   ├── pages/
    │   │   │   │   │   │   ├── test_resource.py

```python
# Tests for "GET /api/books/" and "POST /api/books/"
# File at "project_dir/app/tests/endpoints/api/books/test_resource.py"
from rest_framework.test import APITestCase
from rest_framework import status
from app.factories import BookFactory, UserFactory, TokenFactory


class TestGet(APITestCase):
    def test_when_user_logged_in_has_books_returns_its_books(self):
        user = UserFactory()
        book = BookFactory(user=user)
        BookFactory()  # Another user's book
        token = TokenFactory(user=user)
        self.client.credentials(HTTP_AUTHORIZATION=f'Token {token.value}')

        response = self.client.get('/api/books/')

        expected_data = [
            {
                'id': book.id,
                'title': book.title,
                'author': book.author,
                'year': book.year,
            },
        ]
        self.assertListEqual(
            [response.status_code, response.data],
            [status.HTTP_200_OK, expected_data],
        )


class TestPost(APITestCase):
    def test_when_valid_data_is_sent_returns_created_status(self):
        data = {
            'title': 'The Lord of the Rings',
            'author': 'J.R.R. Tolkien',
            'year': 1954,
        }

        response = self.client.post('/api/books/', data)

        self.assertListEqual(
            [response.status_code, response.data],
            [status.HTTP_201_CREATED, {'id': 1, **data}]
        )
```

In case of nested resources, e.g. `pages`, it should look like this:

```python
# Tests for "PATCH /api/books/{book_id}/pages/" and "DELETE /api/books/{book_id}/pages/"
# File at "project_dir/app/tests/endpoints/api/books/pages/test_resource.py"
from rest_framework.test import APITestCase
from app.factories import BookFactory
from django.db.models import Q


class TestPatch(APITestCase):
    def test_when_valid_data_is_sent_returns_ok_status(self):
        book = BookFactory()
        data = [
            {'content': 'Once upon a time...'},
            {'content': 'The end.'},
        ]

        self.client.patch(f'/api/books/{book.id}/pages/', data)

        qs_filter = Q()
        for item in data:
            qs_filter |= Q(**item)

        self.assertEqual(book.pages.filter(qs_filter).count(), len(data))


class TestDelete(APITestCase):
    def test_when_page_exists_returns_no_content_status(self):
        book = BookFactory()

        self.client.delete(f'/api/books/{book.id}/pages/')

        self.assertEqual(book.pages.count(), 0)
```

[simpletestcase-django-doc]: https://docs.djangoproject.com/en/5.0/topics/testing/tools/#simpletestcase
[testcase-django-doc]: https://docs.djangoproject.com/en/5.0/topics/testing/tools/#testcase
[apitestcase-drf-doc]: https://www.django-rest-framework.org/api-guide/testing/#api-test-cases
[arrange-act-assert-panda]: https://automationpanda.com/2020/07/07/arrange-act-assert-a-pattern-for-writing-good-tests/#The_Pattern
[arrange-act-assert-c2]: https://wiki.c2.com/?ArrangeActAssert
