# Django REST in _Style_

This style guide was built on top of the following versions:

- Django: 3.0.6
- DRF: 3.11.0

## Apps

Your Django app should have 2 "apps" only:

- `core`: Should contain:
  - Settings
  - Overrides of built-in management commands
  - Infrastructure tests (for single server instance)
  - Helpers, utilities and global overrides for infrastructure (functions/classes)

- `app`: Should contain:
  - Views
  - Models
  - Serializers
  - Locales (translations)
  - Routers definitions
  - Source of URLs in settings
  - Custom management commands
  - Helpers and utilities for application (functions/classes)

**Note**: Avoid creating models in the `core` app.

The "why" lies in the section _**Be careful about "applications"**_
from [this article][be-careful-apps-link].

## Tests

### Unit

Normally, unit tests inherit from "[django.test.SimpleTestCase][simpletestcase-django-doc]", as this class doesn't
setup a test database

#### Function

Reproduce the test location and class name as the example below:

```python
# Function "split_names" at "project_dir/app/utils.py"
# Test at "project_dir/app/tests/utils/test_split_names.py"
from django.test import SimpleTestCase

from app.utils import split_names


class TestFunction(SimpleTestCase):
    def test_when_string_without_spaces_is_sent_returns_list_with_one_element(self):
        value = split_names('name')
        # assertions
```

#### Class

Reproduce the test location and class name as the example below:

```python
# Class "AddressSerializer" at "project_dir/app/serializers.py"
# Test at "project_dir/app/tests/serializers/test_address_serializer.py"
from django.test import SimpleTestCase

from app.serializers import AddressSerializer


class TestClass(SimpleTestCase):
    def test_when_instance_is_created_right_properties_are_applied(self):
        data = {'street': 'Test Aloha'}
        serializer_instance = AddressSerializer(data=data)
        # assertions
```

#### Class' methods

Reproduce the test location and class name as the example below:

```python
# Class "Parser", method "to_json" at "project_dir/app/utils.py"
# Test at "project_dir/app/tests/utils/test_parser.py"
from django.test import SimpleTestCase

from app.utils import Parser


class TestToJson(SimpleTestCase):
    def test_when_json_string_is_valid_returns_dictionary(self):
        value = Parser().to_json('{"key": "value"}')
        # assertions
```

### Integration

These can inherit either from "[rest_framework.test.APITestCase][apitestcase-drf-doc]" (for API calls) or
"[django.test.TestCase][testcase-django-doc]"

#### Web API

Picture a view set setting up GET and POST handlers for a resource
`books`. The tests should look like this:

```python
# Test at "project_dir/app/tests/integration/api/books/test_resource.py"
from rest_framework.test import APITestCase


class TestPost(APITestCase):
    def test_when_valid_data_is_sent_returns_created_status(self):
        data = # Data to create a book
        response = self.client.post('/api/books/', data)
        # assertions


class TestGet(APITestCase):
    def test_when_user_logged_in_has_books_returns_its_books(self):
        response = self.client.get('/api/books/')
        # assertions
```

In case of nested resources, e.g. `pages`, it should look like this:

```python
# Test at "project_dir/app/tests/integration/api/books/pages/test_resource.py"
from rest_framework.test import APITestCase


class TestPatch(APITestCase):
    def test_when_valid_data_is_sent_returns_ok_status(self):
        book_id = 1
        data = # Data to update a page
        response = self.client.patch(f'/api/books/{book_id}/pages/', data)
        # assertions


class TestDelete(APITestCase):
    def test_when_page_exists_returns_no_content_status(self):
        book_id = 1
        response = self.client.delete(f'/api/books/{book_id}/pages/')
        # assertions
```

[be-careful-apps-link]: https://blog.doordash.com/tips-for-building-high-quality-django-apps-at-scale-a5a25917b2b5
[simpletestcase-django-doc]: https://docs.djangoproject.com/en/3.0/topics/testing/tools/#simpletestcase
[testcase-django-doc]: https://docs.djangoproject.com/en/3.0/topics/testing/tools/#testcase
[apitestcase-drf-doc]: https://www.django-rest-framework.org/api-guide/testing/#api-test-cases
