# pytest 101

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

It's that simple.

### Configuration

You can pass configs as flags directly into `pytest` command when running, or for convenience, declare them inside *a config file*.

pytest supports `toml` file and `ini` file as config, but `toml` is recommended.

Config values to keep in mind will be shown later on in this guide. The full list can be found on [pytest docs site](https://docs.pytest.org/en/stable/reference/customize.html).

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

## Other frequently used tools

pytest also provides a number of utilities to make writing tests easier.

### Compare floating-point values

When testing floating point values, since they are not exact and can vary a bit, we usually need to approximate the values in some way.

pytest provides `pytest.approx()` as the tool for approximating floating point values, and works with scalars, lists, and NumPy arrays.

```py
import pytest

def test_sum():
    assert (0.1 + 0.2) == pytest.approx(0.3)
```

### Builtin fixtures - Request temp folder for functional tests

pytest provides Builtin **fixtures/function** arguments to request arbitrary resources.

To use a fixture, put its name in the test function parameter list.

One useful one is `tmp_path`, to get a unique temporary directory:

```py
def test_needsfiles(tmp_path):
    # use folder at tmp_path here
    print(tmp_path)
    assert 0
```

There are also other fixtures, and you can write your own too. To show all fixtures, both builtin and custom, run:

```bash
pytest --fixtures 
```

## But what is fixtures?

In testing, a fixture provides a defined, reliable and consistent context for the tests. This could include environment (for example a database configured with known parameters) or content (such as a dataset).

Fixtures define the steps and data that constitute the *arrange* phase of a test. In pytest, they are **functions** you define that serve this purpose. They can also be used to define a test’s *act* phase; this is a powerful technique for designing more complex tests.

The services, state, or other operating environments set up by fixtures are accessed by test functions through arguments. For each fixture used by a test function there is typically a parameter (named after the fixture) in the test function’s definition.

We can tell pytest that a function is a fixture by decorating it with `@pytest.fixture`. Here’s a simple example of what a fixture in pytest might look like:

```py
import pytest

class Fruit:
    def __init__(self, name):
        self.name = name

    def __eq__(self, other):
        return self.name == other.name

@pytest.fixture
def my_fruit():
    return Fruit("apple")

@pytest.fixture
def fruit_basket(my_fruit):
    return [Fruit("banana"), my_fruit]

def test_my_fruit_in_basket(my_fruit, fruit_basket):
    assert my_fruit in fruit_basket
```

## `mark` your test - metadata for tests

Another feature of pytest is `pytest.mark`. `mark` is a helper to set metadata on test functions. Basically attributes for your test cases. With marks, you can:

- Use with plugins
- Select tests when testing with `-m`

Let's talk about how to create one first, and then how to use it to select tests

### What is a `mark`?

**Marks are just `@pytest.mark.*` attributes** on tests or test classes. `*` can be **either a built-in mark, or a custom, registered mark**.

For example, to apply a mark named `web_integration_tests` on the `test_send_http` test case, we would first register the mark's name inside our config file (or config hook), with everything after the `:` as comments for that mark:

```toml
[pytest]
markers = [
    "web_integration_tests", # our custom mark
    "slow: marks tests as slow (deselect with '-m \"not slow\"')", # a custom mark with documentations after :
]
```

If you don't register the mark, it will emits a warning to avoid mistyped marks (thanks python and dynamic typing), and that mark won't have any effect. Of course, except from the built-in ones.

After that, you can apply that mark on the test case like this:

```python
import pytest

@pytest.mark.web_integration_tests
def test_send_http():
    pass  # perform some webtest test for your app
```

Now that case is marked! Onto using it.

### Selecting tests

Let's say you have a project with tests like this (would be in different files):

```python
import pytest

@pytest.mark.web_integration_tests
def test_send_http():
    pass  # perform some webtest test for your app

# other tests below, unmarked or marked with different marks
@pytest.mark.serial
def test_another():
    pass

class TestClass:
    def test_method(self):
        pass
```

You have test cases that is doing integration tests and want to run them only. In that case, you can use `-m` to select and run only these tests:

```bash
pytest -m web_integration_tests
```

Or, you can run all tests except the integrations tests with:

```bash
pytest -m "not web_integration_tests"
```

Usual python-style boolean expressions can be used here (and, or, not, parentheses). Go wild with the marks selector.

```bash
pytest -m "serial and (bucket or database or not mockup)"
```

Usage with plugins will be covered in, well, plugins.

### Built-in marks

pytest also includes a few built-in marks to use, and they will affect the test case marked. Here's a few of them that exists but is quite dumb (at least imo):

- usefixtures - use fixtures on a test function or class. In case you want to apply fixtures to a whole class. Don't use this for the test function, that would be dumb.
- filterwarnings - filter certain warnings of a test function. *Why tho?* But it's there if you need it.

Now we get to the useful ones:

- skip - always skip a test function
- skipif - skip a test function if a certain condition is met
- xfail - produce an “expected failure” outcome if a certain condition is met
- parametrize - perform multiple calls to the same test function.

### Skipping tests

#### `skip` mark

To skip tests for whatever reasons, usually OS-dependent tests or similar situations, you can use the `skip` mark, optionally with a reason:

```python
# why tho
@pytest.mark.skip(reason="no way of currently testing this")
def test_the_unknown(): ...
```

#### calling `pytest.skip()` function

Aside from using the `skip` mark, you can call `pytest.skip(reason)` function directly:

```py
def test_function():
    if not valid_config():
        pytest.skip("unsupported configuration")
```

For the (imho) **only reasonable reason** for a full skip is for **platform-dependent tests**, which applies to the **whole module/file**. This should look like *this*, for example:

```py
import sys

import pytest

if not sys.platform.startswith("win"):
    pytest.skip("skipping windows-only tests", allow_module_level=True)
```

Remember not to forget the `allow_module_level=True` args or chaos will ensue.

#### `skipif` fixtures

If you really, really, really want to use fixtures to keep code inside test cases clean or something, you can use `skipif` fixture to only skip on a True condition. 

For example, to skip a test on python version lower than 3.13:

```py
import sys

@pytest.mark.skipif(sys.version_info < (3, 13), reason="requires python3.13 or higher")
def test_function(): ...
```

Also, *did you know*, to share a similar attribute, you can assign it to a variable?

```py
minversion = pytest.mark.skipif(
    version < (1, 1), reason="at least mymodule-1.1 required"
)

@minversion
def test_function(): ...
```

*insert the more you know meme*

### Parameterizes your test cases - `@pytest.mark.parametrize`

If your test cases have similar code but just different input/output, or you want to run a test case on a set of inputs, you dont need to re-write that case many time, just use `@pytest.mark.parametrize`

For example, to run a test case with input arg `a` using 5 different values, `0,999,1.0,2000,-0.0` for it, all expecting the case assert to pass.

```py
import pytest

@pytest.mark.parametrize("test_input", [0,999,1.0,2000,-0.0])
def test_abs(test_input):
    assert abs(test_input) >= 0
```

Or, another frequent use case, a list of inputs and expected result for each input set. For example, testing an `add(a, b)` function with a, b and expected. In case of many args, each arg will be a tuple `(value1, value2, ...)`.

```py
import pytest

@pytest.mark.parametrize("a,b,expected", [
    (1,2,3),
    (1,3,4),
    (-2,2,0)
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

This helps writing cases for techniques like *boundary value analysis* and *equivalence partitioning* **a lot faster and less repetitive**.

However, **keep in mind**, parameter values are **passed as-is to tests** (no copy whatsoever).

For example, if you pass a list or a dict as a parameter value, and the test case code mutates it, the mutations will be reflected in subsequent test case calls.

Why doesn't it make a deep copy, to "ensure the integrity of tests"? That I do not know. *It boggles me that they leave out such an important details.* They know about it, yet refuses to change.

## Tests structure

### Test discovery

pytest does **NOT** have a fixed, required folder structure.
You can put tests anywhere you want, in any project organization as needed.

Instead of fixed "test projects" like JUnit, NUnit,..., pytest uses the standard convention for "Python test discovery" instead:

- If no arguments are specified then collection starts from testpaths (if configured) or the current directory. Alternatively, command line arguments can be used in any combination of directories, file names or node ids.

- Recurse into directories, unless they match `norecursedirs` (used for stuff like `__pycache__`, etc to save time).

- In those directories, search for `test_*.py` or `*_test.py` files, imported by their [test package name](https://docs.pytest.org/en/stable/explanation/goodpractices.html#test-package-name).

- From those files, collect test items:

  - `test` prefixed test functions or methods outside of class.

  - `test` prefixed test functions or methods inside `Test` prefixed test classes (without an `__init__` method). Methods decorated with `@staticmethod` and `@classmethods` are also considered.

As long as your test files and test classes/functions are named like the convention, it will be discovered. You can also change the convention if needed with a **configuration file**.

### WARNING

Even though pytest can **discover** the tests, it cannot automatically handle **imports** depends on your project layout because Python shennanigans. Some examples are given below.

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
