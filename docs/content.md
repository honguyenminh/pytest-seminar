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

As long as your test files and test classes/functions are named like the convention, it will be discovered. You can also change the convention if needed with a **configuration file**.

### WARNING

Even though pytest can **discover** the tests, it cannot automatically handle **imports** depends on your project layout because Python shennanigans. Some example are given below.

### Separate test folder

You can have the tests as a separate package/folder from your app code which is wrapped in a `src` folder:

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

If you use this style, where there's a `src` folder wrapping your app code, auto import will fail, and you will need to do extra shennanigans below.

Why deal with this hassle and just not use a `src` wrapping folder? [This is why.](https://blog.ionelmc.ro/2014/05/25/python-packaging/#the-structure%3E)

TLDR, to ensure tests are able to run with installed versions of your app, since current working folder is auto added into python import path and it's unsafe to rely on that behavior.

Either invoke pytest with an environment variable:

```bash
PYTHONPATH=src pytest
```

Or, better yet, add this to your `pytest.toml` config file:
```toml
[pytest]
pythonpath = ["src"]
```
Then just run `pytest`. Your tests import should be able to work now.

### Inlined tests

Another structure is to have tests as part of the application code, if you want to distribute tests with your app:

```
pyproject.toml
[src/]mypkg/
    __init__.py
    app.py
    view.py
    tests/
        __init__.py
        test_app.py
        test_view.py
        ...
```

Then, just run your tests with:
```bash
pytest --pyargs src/mypkg
```
Replace `src/mypkg` with the path to your package from current directory.

### EXTRA: Import mode