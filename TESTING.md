# Testing Guidelines

## Table of Contents

- [Test naming](#test-naming)
- [Test assertions](#test-assertions)
- [Test structure: Arrange-Act-Assert](#test-structure-arrange-act-assert)
- [Unit Tests](#unit-tests)
  - [Testing functions and classes](#testing-functions-and-classes)
- [Integration](#integration)
  - [Web API](#web-api)

## Test naming

"Proposition" is the pattern to be followed for test names and should be readable as a sentence.

Default behaviors or no conditions should look like `test_that_it_[expected outcome]`, e.g.:
`test_that_it_saves_the_user`.

Conditions should look like `test_when_[conditions]_then_[expected outcome]`, e.g.:
`test_when_active_flag_is_true_and_user_has_access_then_returns_active_users`.

Each test should assert "and" conditions only. The "or" conditions are represented by multiple tests, like so:

```
test_when_active_flag_is_true_then_returns_active_users

test_when_active_flag_is_not_set_then_returns_active_users
```

## Test assertions

Tests should only have one assertion. This is a side effect of using propositions as a naming convention.

If the test is asserting a single concept that requires multiple assertions then go for it, but having multiple
assertions potentially requires multiple runs to fully fix a broken test.

Bundling assertions in a single call of `assertListEqual`, `assertDictEqual` etc. is an acceptable way to break this
rule and this is what assertion abstractions should be based on. Here's an example of a very common assertion
abstraction:

```python
import json

from rest_framework import status
from rest_framework.renderers import JSONRenderer
from rest_framework.response import Response
from rest_framework.test import APITestCase as DRFAPITestCase


def _extract_response_data(response: Response):
    """
    Extracts a diff-friendly response data from a DRF Response object.
    """
    if not hasattr(response, "data"):
        return response.content
    return json.loads(JSONRenderer().render(response.data))


class APITestCase(DRFAPITestCase):
    def assertResponse(self, response: Response, status_code: int, expected_data) -> None:
        self.assertListEqual(
            [response.status_code, _extract_response_data(response)],
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

In production code, principles like DRY (Don't Repeat Yourself) are essential for maintaining consistency and making
changes that have a wide effect on the codebase with the least amount of effort.

The Arrange-Act-Assert pattern (a.k.a. AAA or Triple-A) promotes clarity, isolation, and ease of debugging in test code
by providing a clear and systematic framework for organizing tests.

It consists of three phases.
- In the `Arrange` phase, you set up the preconditions and context for the test.
- The `Act` phase involves executing the specific action or behavior being tested.
- In the `Assert` phase, you verify that the outcome matches the expected result.

Although setting up for each test can impact test speed negatively, it ensures that tests are straightforward to fix
when issues arise.

This can also be seen as "learning" tests, where the test is a documentation of how the product/code should work. This
is particularly useful for new team members or when a team member hasn't touched that particular area of the codebase in
a while.

This is a simple example of some test cases with DRY in mind (try to fully understand the code as you read):

```python
# A test file for an endpoint
from app.tests.utils import add_book_to_user, TestAssertionsMixin, TestLoggedIn
from rest_framework import status


class TestGet(TestLoggedIn, TestAssertionsMixin):
    def setUp(self):
        super().setUp()
        self.book = add_book_to_user(self.user)

    def test_when_user_is_not_logged_in_then_returns_unauthorized(self):
        self.logout_user()
        response = self.client.get(f'/api/users/{self.user.id}/books/')

        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_when_user_logged_in_has_books_returns_its_books(self):
        response = self.client.get(f'/api/users/{self.user.id}/books/')
        self.assertBookData(response, self.book)
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
    def assertBookData(self, response, book):
        if response.status_code != status.HTTP_200_OK:
            raise AssertionError(
                f'Expected status code 200, got {response.status_code} and response:\n{response.content}'
            )
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

Now the same set of test cases but with the Arrange-Act-Assert pattern (again, try to understand the code as you read):

```python
from uuid import uuid4
from rest_framework.test import APITestCase
from rest_framework import status
from app.factories import BookFactory, UserFactory, TokenFactory


class TestGet(APITestCase):
    def test_when_user_is_not_logged_in_then_returns_unauthorized(self):
        # Arrange
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

The Arrange-Act-Assert steps may be skipped if a step is simple enough, like so:

```python
from django.test import SimpleTestCase


def sum_numbers(*args: int) -> int:
    return sum(args)


class TestSumFunction(SimpleTestCase):
    # With AAA steps
    def test_that_it_returns_correct_sum(self):
        num1, num2 = 5, 3

        result = sum_numbers(num1, num2)

        self.assertEqual(result, 8)

    # Skipping AAA steps
    def test_that_it_returns_correct_sum(self):
        self.assertEqual(sum_numbers(5, 3), 8)
```

## Unit Tests

Normally, unit tests inherit from [`django.test.SimpleTestCase`][simpletestcase-django-doc] as this class doesn't make
use of the test database.

### Testing functions and classes

Each function should have a test of its own called `Test<Function>` and each class method should have a test of its own
called `Test<Class>`.

Underscores (`_`) should not be taking into account for test names (e.g. testing `__init__` would have `TestInit`).

Any class, method or function beginning with a single underscore (`_`) should be tested or mocked (exception for when
there's no other way). They are considered private and should only be tested indirectly.

For the test case examples on this section, take into account the following code and files:

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

The test folder structure for the code above should look like this:

    project_dir/
    └── app/
        ├── utils.py
        ├── serializers.py
        └── tests/
            ├── utils/
            │   ├── test_split_names.py
            │   └── test_parser.py
            └── serializers/
                └── test_address_serializer.py

```python
# Test at "project_dir/app/tests/utils/test_split_names.py"
from django.test import SimpleTestCase

from app.utils import split_names


class TestFunction(SimpleTestCase):
    def test_when_string_without_spaces_is_sent_returns_list_with_one_element(self):
        value = split_names('string')

        self.assertListEqual(value, ['string'])

# Test at "project_dir/app/tests/serializers/test_address_serializer.py"
from django.test import SimpleTestCase

from app.serializers import AddressSerializer


class TestInit(SimpleTestCase):
    def test_when_instance_is_created_then_right_properties_are_applied(self):
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
    def test_when_json_string_is_valid_then_returns_dictionary(self):
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
    └── app/
        ├── factories.py
        ├── models.py
        ├── serializers.py
        └── tests/
            └── endpoints/
                └── api/
                    └── books/
                        ├── test_resource.py
                        └── pages/
                            └── test_resource.py

```python
# Tests for "GET /api/books/" and "POST /api/books/"
# File at "project_dir/app/tests/endpoints/api/books/test_resource.py"
from rest_framework.test import APITestCase
from rest_framework import status
from app.factories import BookFactory, UserFactory, TokenFactory


class TestGet(APITestCase):
    def test_when_user_logged_in_has_books_then_returns_its_books(self):
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
    def test_when_valid_data_is_sent_then_returns_created_status(self):
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
    def test_when_valid_data_is_sent_then_returns_ok_status(self):
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
    def test_when_page_exists_then_returns_no_content_status(self):
        book = BookFactory()

        self.client.delete(f'/api/books/{book.id}/pages/')

        self.assertEqual(book.pages.count(), 0)
```

[simpletestcase-django-doc]: https://docs.djangoproject.com/en/5.0/topics/testing/tools/#simpletestcase
[testcase-django-doc]: https://docs.djangoproject.com/en/5.0/topics/testing/tools/#testcase
[apitestcase-drf-doc]: https://www.django-rest-framework.org/api-guide/testing/#api-test-cases
[arrange-act-assert-panda]: https://automationpanda.com/2020/07/07/arrange-act-assert-a-pattern-for-writing-good-tests/#The_Pattern
[arrange-act-assert-c2]: https://wiki.c2.com/?ArrangeActAssert
