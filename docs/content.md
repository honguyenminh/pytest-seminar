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

## Example test files

A test file would look something like:

```py
# func would be imported
def func(x):
    return x + 1

def test_answer():
    assert func(3) == 5
```

If extra functionality is needed, import `pytest`. For example, to test for a raised exception:

```py
import pytest

def f():
    raise SystemExit(1)

def test_mytest():
    with pytest.raises(SystemExit):
        f()
```

You can also group tests into classes starting with `Test`, to contain the test cases. No need to subclass from anything, just normal class.

```py
class TestClass:
    def test_one(self):
        x = "this"
        assert "h" in x

    def test_two(self):
        x = "hello"
        assert hasattr(x, "check")
```

Grouping into classes like this helps with:

- Test organization
- Sharing fixtures (stuff created to use for testing) for tests only in that particular class
- Applying marks at the class level and having them implicitly apply to all tests

Something **to be aware of** when grouping tests inside classes is that **each test has a unique instance of the class**. Having each test share the same class instance would be bad for test isolation and would promote poor test practices. To demonstrate:

```py
class TestClassDemoInstance:
    value = 0

    def test_one(self):
        self.value = 1
        assert self.value == 1

    def test_two(self):
        assert self.value == 1
```

This would fail at test_two.  
However, attributes on the class itself is shared since instances share the same class. You usually dont need to care about this though.


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

For historical reasons, pytest defaults to the `prepend` import mode, but it's recommend to use the `importlib` import mode for new projects instead.

**TLDR**: Use `importlib` mode to have tests with the same name.

Add this to your config file:

```toml
[pytest]
addopts = ["--import-mode=importlib"]
```

#### Nerdy explanation

Since there are no packages to derive a full package name from, pytest will import your test files as *top-level* modules. The test files in the first example ([src layout](#separate-test-folder)) would be imported as `test_app` and `test_view` top-level modules by adding tests/ to sys.path.

This results in a drawback compared to the import mode `importlib`: **your test files must have unique names**.

If you need to have **test modules with the same name**, as a workaround you might add __init__.py files to your tests folder and subfolders, changing them to packages:

```
pyproject.toml
mypkg/
    ...
tests/
    __init__.py
    foo/
        __init__.py
        test_view.py
    bar/
        __init__.py
        test_view.py
```

Now pytest will load the modules as `tests.foo.test_view` and `tests.bar.test_view`, allowing you to have modules with the same name. But now this introduces a subtle problem: in order to load the test modules from the `tests` directory, pytest prepends the root of the repository to sys.path, which adds the side-effect that now `mypkg` is also importable.

This is problematic if you are using a tool like `tox` to test your package in a virtual environment, because you want to test the installed version of your package, not the local code from the repository.

