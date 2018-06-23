# Django REST Style Guide

This style guide refers to the following versions:

- Django: 2.0.5
- DRF: 3.8.2

## Tests

Inside each app, you should have a module named `tests`, and inside
it you'll have `integration` and `unit` modules, which your tests
should be distributed into, in files and/or more modules.

### Integration

#### API

Picture a view set setting up GET and POST handlers for a resource
`books`. Create a module under `integration` with the name of the
resource, where you should put a `test_resource.py` with the tests
in it. The tests should look like this:

```python
# project_dir/app/tests/integration/books/test_resource.py
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
# project_dir/app/tests/integration/books/pages/test_resource.py
from rest_framework.test import APITestCase

class TestPatch(APITestCase):
    def test_when_valid_data_is_sent_returns_ok_status(self):
        book_id = 1
        data = # Data to update a page
        response = self.client.patch('/api/books/{}/pages/'.format(book_id), data)
        # assertions


class TestDelete(APITestCase):
    def test_when_page_exists_returns_no_content_status(self):
        book_id = 1
        response = self.client.delete('/api/books/{}/pages/'.format(book_id))
        # assertions
```
