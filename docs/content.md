# Pytest 101

*pytest* is a tool/framework used for writing and executing tests in Python

pytest includes:
- the tool binary for executing tests
- library for writing tests
- additional "fixtures" and "plugins" (explained later)

## Getting started

Install the tooling and library with PyPI:

```bash
pip install pytest
```

Or add to **DEVELOPMENT** requirements.txt.

Then, run the command `pytest` in your project root folder to try running it.

## Tests structure

### Test discovery

Pytest does **NOT** have a fixed, required folder structure.
You can put tests anywhere you want, in any project organization as needed.

Instead of fixed "test projects" like JUnit, NUnit,..., pytest uses the standard convention for "Python test discovery" instead:

- If no arguments are specified then collection starts from testpaths (if configured) or the current directory. Alternatively, command line arguments can be used in any combination of directories, file names or node ids.

- Recurse into directories, unless they match `norecursedirs` (used for stuff like `__pycache__`, etc to save time).

- In those directories, search for `test_*.py` or `*_test.py` files, imported by their [test package name](https://docs.pytest.org/en/stable/explanation/goodpractices.html#test-package-name).

- From those files, collect test items:

  - test prefixed test functions or methods outside of class.

  - test prefixed test functions or methods inside Test prefixed test classes (without an __init__ method). Methods decorated with @staticmethod and @classmethods are also considered.

As long as your test files and test classes/functions are named like the convention, it will be discovered. You can also change the convention if needed with a **configuration file**

You can have the tests as a separate package from your app code:
```
pyproject.toml
src/
    mypkg/
        __init__.py
        app.py
        view.py
tests/
    test_app.py
    test_view.py
    ...
```

