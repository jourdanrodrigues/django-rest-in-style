# Testing Guidelines

## Table of Contents

- [Glossary](#glossary)
- [Test suite assumptions](#test-suite-assumptions)
- [Test suite priorities](#test-suite-priorities)
- [Naming rule](#naming-rule)
- [Assertion rule](#assertion-rule)
- [Test case structure rule: Arrange-Act-Assert](#test-case-structure-rule-arrange-act-assert)
- [Unit vs Integration tests](#unit-vs-integration-tests)
  - [Tests that don’t need database](#tests-that-dont-need-database)
  - [Tests that need database](#tests-that-need-database)
  - [Tests covering HTTP endpoints](#tests-covering-http-endpoints)
- [Testing rules for functions and classes](#testing-rules-for-functions-and-classes)
- [Testing rules for HTTP endpoints](#testing-rules-for-http-endpoints)
  - [Test file path structure](#test-file-path-structure)
  - [Test class names](#test-class-names)
  - [Full payload test cases](#full-payload-test-cases)
  - [Query count test cases](#query-count-test-cases)
- [Mocking rule](#mocking-rule)
  - [Prohibited from mocking](#prohibited-from-mocking)
  - [When can we mock](#when-can-we-mock)
  - [What can we mock](#what-can-we-mock)
  - [How can we mock](#how-can-we-mock)
    - [Patching via decorators and context managers](#patching-via-decorators-and-context-managers)

## Glossary

- Test class: A holder of test cases for a test subject
- Test case: A script aiming to preserve one specific behavior of a test subject
- Behavior: Observable output or interaction of the test subject for given inputs, exceptions or settings
    - This does NOT mean **implementation**, which is **how** a test subject returns an output.
- Entity: A function, a class or a class method

## Test suite assumptions

- A test file is rarely read in its entirety. Devs tend to read tests once they break, one by one.
- Human brains can’t hold many abstraction contexts at a time. This means keeping in mind the contexts of a call chain: `function → class method → another function`.

## Test suite priorities

These are not to be interpreted as rules. Their intention is to drive the rules and recommendations in this document.

A test case should:

1. Maintain production behavior
2. Be readable - ideally serving as documentation
3. Be self-contained - have the least amount of code navigation as possible
4. Be consistent - i.e. not flaky
5. Be fast - e.g. only create data required to make sure behavior is maintained

## Naming rule

A [philosophical proposition](https://en.wikipedia.org/wiki/Proposition) is the pattern to be followed for test names and should be, as much as possible, readable as a sentence.

Default behaviors or no conditions must look like `test_that_it_[expected_outcome]`, e.g.: `test_that_it_saves_the_user`.

Conditions must look like `test_when_[conditions]_then_[expected_outcome]`, e.g.: `test_when_active_flag_is_true_and_user_has_access_then_returns_active_user`.

Each test must assert "and" conditions only. The "or" conditions are represented by multiple tests, like so:

```
test_when_active_flag_is_true_then_returns_active_users

test_when_active_flag_is_not_set_then_returns_active_users
```

## Assertion rule

Each test case must only preserve one behavior, and therefore, contain only one assertion.

An exception is when a single concept requires multiple assertions. The caveat is that it potentially requires multiple runs to fix a broken test, which is not ideal from a dev perspective.

Bundling assertions with `assertListEqual`, `assertDictEqual` etc. is an acceptable way to break this rule without having multiple assertions. This is what assertion abstractions should be based on. Here's an example of a very common assertion abstraction:

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

## Test case structure rule: Arrange-Act-Assert

In production code, principles like DRY (Don't Repeat Yourself) are essential for maintaining consistency and making changes that have a wide effect on the codebase with the least amount of effort.

The Arrange-Act-Assert pattern promotes readability, isolation, and ease of debugging in test code by making it self-contained with the least amount of abstractions, so they end up WET instead (Writing Every Time).

It consists of three phases:

- In the `Arrange` phase, you set up the preconditions and context for the test.
- The `Act` phase involves executing the specific action or behavior being tested.
- In the `Assert` phase, you verify that the outcome matches the expected result.

Although setting up for each test can impact test speed negatively, it ensures that each test has only the setup required for the behavior it aims to maintain.

This can also be seen as "learning" tests, where they become documentation. This is particularly useful for new team members or when a team member hasn't touched that particular area of the codebase in a while.

This is a simple example of some test cases with DRY in mind (to be considered *bad*):

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

    def test_when_user_logged_in_has_books_then_returns_its_books(self):
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

Now, the same set of test cases but with the Arrange-Act-Assert pattern (to be considered *good*):

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

    def test_when_user_logged_in_has_books_then_returns_its_books(self):
        # Arrange
        user = UserFactory()
        book = BookFactory()
        user.books.add(book)
        token = TokenFactory(user=user)
        self.client.credentials(HTTP_AUTHORIZATION=f'Token {token.value}')

        # Act
        response = self.client.get(f'/api/users/{user.id}/books/')

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

## Unit vs Integration tests

Since it’s hard to draw a line on what are unit and integration tests, we just think of them by what they require to work.

### Tests that don’t need database

These must inherit from [django.test.SimpleTestCase](https://docs.djangoproject.com/en/5.0/topics/testing/tools/#simpletestcase) (or a subclass of it) so it doesn’t setup the database at all, making it extremely fast.

Model instances can be created without existing in the database like so:

```python
user = User(email="email@example.com")

do_something_with_user_email(user)
```

This way we can test code that makes use of model instances but don’t perform any database operations.

### Tests that need database

These must inherit from [django.test.TestCase](https://docs.djangoproject.com/en/5.0/topics/testing/tools/#testcase) (or a subclass of it) so it setups what’s necessary to have database operations. Data setup should be made through factories.

### Tests covering HTTP endpoints

These must inherit from [one of DRF’s test cases](https://www.django-rest-framework.org/api-guide/testing/#api-test-cases) (or subclasses of them) because they have a better client for testing purposes.

They have their own versions of `SimpleTestCase`, `TransactionTestCase` etc.

## Testing rules for functions and classes

Each function should have a test class of its own called `TestFunction` and each class method should have a test class of its own called `Test<MethodName>`, both in Pascal case.

Underscores should not be taken into account for test class names (e.g. testing `__init__` would have `TestInit`).

Any class, method or function with a leading single underscore must NOT be tested directly. They are considered private and should only be tested indirectly. If there’s a need to test these, first expose them by removing the leading underscore.

Take into account the following code and files:

```python
# File at "project_dir/app/utils.py"
import json

def split_names(full_name: str) -> list[str]:
    return full_name.split(' ')

class Parser:
    @staticmethod
    def to_json(json_string: str) -> dict:
        return json.loads(json_string)
```

```python
# File at "project_dir/app/serializers.py"
from rest_framework import serializers

class AddressSerializer(serializers.Serializer):
    street = serializers.CharField()
```

The test folder structure for the code above should look like this:

```
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
```

```python
# Test at "project_dir/app/tests/utils/test_split_names.py"
from django.test import SimpleTestCase

from app.utils import split_names

class TestFunction(SimpleTestCase):
    def test_when_string_without_spaces_is_sent_then_returns_list_with_one_element(self):
        value = split_names('string')

        self.assertListEqual(value, ['string'])
```

```python
# Test at "project_dir/app/tests/serializers/test_address_serializer.py"
from django.test import SimpleTestCase

from app.serializers import AddressSerializer

class TestInit(SimpleTestCase):
    def test_when_instance_is_created_then_right_properties_are_applied(self):
        data = {'street': 'Test Aloha'}

        # Serializers are an exception because they have a bulky API, so these 3 lines are part of the "act"
        serializer = AddressSerializer(data=data)
        serializer.is_valid(raise_exception=True)
        instance = serializer.save()

        self.assertEqual(instance.street, data['street'])
```

```python
# Test at "project_dir/app/tests/utils/test_parser.py"
from django.test import SimpleTestCase

from app.utils import Parser

class TestToJson(SimpleTestCase):
    def test_when_json_string_is_valid_then_returns_dictionary(self):
        value = Parser().to_json('{"key": "value"}')

        self.assertDictEqual(value, {'key': 'value'})
```

## Testing rules for HTTP endpoints

### Test file path structure

Endpoint tests must live under an `endpoints` folder and have the endpoint path be mimicked by the folder structure, with a `test_resource.py` file to hold the tests for that endpoint.

This approach detaches the tests from the actual code, but the goal here is to test endpoint behavior, not the code that implements it.

A valid concern for this is that one would lose track of which endpoint handlers are tested. This should be mitigated by having an easily accessible test coverage report to expose untested code.

The test structure for the covering endpoints like `GET /api/books/` and `PUT /api/books/<book_id>/pages/<page_id>/` should be:

```
project_dir/
└── app/
    ├── ...
    └── tests/
        └── endpoints/
            └── api/
                └── books/
                    ├── test_resource.py
                    └── pages/
                        └── test_resource.py
```

### Test class names

The names will be based on HTTP methods, e.g. a `GET /api/books/` will have a `TestGet` to hold its test cases, and a `PATCH /api/books/<id>/` will have a `TestPatch`.

Parameters in detail endpoints (e.g. a resource ID) should not be taken into account when naming test classes, except for when there’s a clash like `<METHOD> /api/books/` and `<METHOD> /api/books/<id>`, in which case there should be a `Test<Method>` for the root and `Test<Method>One` for the detail.

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

class TestGetOne(APITestCase):
    def test_that_it_returns_the_book(self):
        user = UserFactory()
        book = BookFactory(user=user)
        token = TokenFactory(user=user)
        self.client.credentials(HTTP_AUTHORIZATION=f'Token {token.value}')

        response = self.client.get(f'/api/books/{book.id}/')

        expected_data = {
            'id': book.id,
            'title': book.title,
            'author': book.author,
            'year': book.year,
        }
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

### Full payload test cases

It's crucial to maintain the API contract via tests. To achieve that, we should include at least one test case per endpoint that asserts the entire payload returned by the API. This safeguards against potential discrepancies like:

- Returning more data than expected
- Presenting data in an unexpected format
- Altering the order of elements
- Omitting information

```python
from rest_framework.test import APITestCase
from rest_framework import status
from app.factories import UserFactory, TokenFactory

class TestGet(APITestCase):
    def test_that_it_returns_expected_response(self):
        user = UserFactory()
        token = TokenFactory(user=user)
        self.client.credentials(HTTP_AUTHORIZATION=f"Token {token.value}")

        response = self.client.get("/api/users/me/")

        expected_data = {
            "id": user.id,
            "username": user.username,
            "email": user.email,
            "first_name": user.first_name,
            "last_name": user.last_name,
            # No password hash
            "permissions": user.get_permissions(),
            "is_staff": user.is_staff,
            "is_active": user.is_active,
            "date_joined": user.date_joined.isoformat().replace('+00:00', 'Z'),
            "last_login": user.last_login.isoformat().replace('+00:00', 'Z'),
        }
        self.assertListEqual(
            [response.status_code, response.data],
            [status.HTTP_200_OK, expected_data],
        )
```

### Query count test cases

Django's ORM is very powerful but can be tricky to use correctly when dealing with larger datasets. Quite often endpoints will return the correct data but will run into a linear or exponential query issue to do so.

In this test case, a larger amount of data should be created to make sure any query issues are exposed.

These are supposed to be the slowest test cases, and there should ideally be only one per endpoint.

```python
from rest_framework.test import APITestCase
from app.factories import UserFactory, TokenFactory, BookFactory

class TestGet(APITestCase):
    def test_that_it_performs_expected_query_count(self):
        for user in UserFactory.create_batch(20):
            books = BookFactory.create_batch(5, user=user)
            user.books.add(*books)

        with self.assertNumQueries(2):
            """
            Captured queries were:
            1. SELECT "users_user".* FROM "users_user" WHERE "users_user"."id" = 1
            2. SELECT "books_book".* FROM "books_book" WHERE "books_book"."user_id" in (1)
            """
            self.client.get("/api/users/")
```

Can be also worth to check for the response status code to make sure the request actually worked.

A good practice is to write this test starting with 0 queries, let the test fail and then adjust to the correct number, also picking up the queries from the message log and placing as a message next to the assertion.

There's a valid concern about this becoming outdated. Creating an abstraction to check for the actual queries might be worthwhile. See an example of one [here](https://github.com/jourdanrodrigues/django-template/blob/c7efb6a745587786507dfbabdeb2dc180bbc64f2/app/tests/utils.py#L25).

## Mocking rule

Mocking should not be done. What we address in this section are the exceptions and their approaches.

### Prohibited from mocking

Any entity or property with a leading underscore (magic methods not included) should NOT be mocked. They are considered private and should only be tested indirectly.

If there’s a need to mock one of those, they should be made public by removing the leading underscore.

Exception to this is when we have the need to mock a dependency entity that’s private.

### When can we mock

We should reach for mocking only when:

- performing network-related operations, like HTTP clients
- triggering behaviors that can’t be reached via input parameters, like raising lib exceptions

### What can we mock

Class and instance properties (including methods) are all that we should mock. It is possible to mock classes and functions themselves, but the way it is done ties implementation.

If the need to mock a function arises, we should:

- in case of our ownership, turn that function into a class method and mock that instead
- in case of a lib:
    - look for its source code and try to spot a class method that could be mocked instead
    - wrap the function in an abstraction class and mock that instead

### How can we mock

First of all, under no circumstance we should have a `settings.TEST` check in ***production code***. It pushes away production environment from test environment by default, potentially hiding issues that tests should be exposing.

Such a flag must be used exclusively for tweaking settings, like using file system instead of S3 for handling files generated during tests.

This should apply to any variant of `settings.TEST` (like `settings.FEATURE_ENABLED`) that’s purely meant for disabling a feature during tests.

Instead of that, we should make use of patches.

#### Patching via decorators and context managers

It’s recommended that we use context managers to tie a mock to the code that’s being tested. Using decorators will apply the mock also to the `Arrange` and `Assert` sections of the test, which could end up making us test the mocks themselves.

```python
from secrets import SystemRandom
from unittest.mock import patch
from django.test import SimpleTestCase

def get_2_random_numbers(numbers: list[int]) -> list[int]:
    return SystemRandom().sample([1, 2, 3], 2)

class TestGet2RandomNumbers(SimpleTestCase):
    def test_that_it_returns_sample_from_random_with_context_manager(self):
        numbers = [4, 5, 6]
        expected_output = SystemRandom().sample(numbers, 2)

        with patch.object(SystemRandom, "sample", return_value=expected_output):
            output = get_2_random_numbers(numbers)

        self.assertListEqual(expected_output, output)

    @patch.object(SystemRandom, "sample", return_value=[7, 8, 9])
    def test_that_it_returns_sample_from_random_with_decorator(self, sample_mock):
        numbers = [4, 5, 6]
        expected_output = SystemRandom().sample(numbers, 2)

        output = get_2_random_numbers(numbers)

        self.assertListEqual(expected_output, output)
```

Both of these test cases will pass successfully, but the second case, with a decorator patch, has the `Arrange` section also mocked, which means the `numbers` variable never get used anywhere. That test is essentially testing mocks.

Of course, this is a very simple scenario and decorator patches obviously have their value, but they should be used with carefulness and is probably better to default to mocking with context managers.
