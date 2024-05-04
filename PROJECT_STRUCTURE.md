# Project Structure

Your project should look roughly like this:

    project_dir/
    ├── app/
    │   ├── __init__.py
    │   ├── locale/
    │   ├── management/
    │   ├── migrations/
    │   ├── tests/
    │   ├── admin.py
    │   ├── apps.py
    │   ├── authentication.py
    │   ├── filters.py
    │   ├── models.py
    │   ├── permissions.py
    │   ├── serializers.py
    │   ├── urls.py
    │   └── views.py
    ├── core/
    │   ├── __init__.py
    │   ├── management/
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
an underscore (`_`) and have what should be "exposed" imported in the `__init__.py`. IDEs don't work well with more than
1 level of nesting, though.

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
