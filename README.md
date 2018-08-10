# pytest-depends

This pytest plugin allows you to declare dependencies between pytest tests, where dependent tests will not run if the
tests they depend on did not succeed.

Of course, tests should be self contained whenever possible, but that doesn't mean this doesn't have good uses.

This can be useful for when the failing of a test means that another test cannot possibly succeed either, especially
with slower tests. This isn't a dependency in the sense of test A sets up stuff for test B, but more in the sense of if
test A failed there's no reason to bother with test B either.

## Installation

Simply install using `pip` (or `easy_install`):

```
pip install pytest-depends
```

## Usage

``` python
BUILD_PATH = 'build'

def test_build_exists():
    assert os.path.exists(BUILD_PATH)

@pytest.depends(on=['test_build_exists'])
def test_build_version():
    result = subprocess.run([BUILD_PATH, '--version', stdout=subprocess.PIPE)
    assert result.returncode == 0
    assert '1.2.3' in result.stdout
```

This is a simple example of the situation mentioned earlier. In this case, the first test checks whether the build file
even exists. If this fails, the other test will not be ran, as there is no point in doing to.

## Order

This plugin will automatically re-order the tests so that tests are run after the tests they depend on. If another
plugin also reorders tests (such as `pytest-randomly`), this may cause problems, as dependencies that haven't ran yet
are considered failures.

This plugin attempts to make sure it runs last to prevent this issue, but there are no guarantees this is successful. If
you run into issues with this in combination with another plugin, feel free to open an issue.

## Naming

There are multiple ways to refer to each test. Let's start with an example, which we'll call `test_file.py`:

``` python
class TestClass(object):
    @pytest.mark.depends(name='foo')
    def test_in_class(self):
        pass

@pytest.mark.depends(name='foo')
def test_outside_class():
    pass

def test_without_name(num):
    pass
```

The `test_in_class` test will be available under the following names:

- `test_file.py::TestClass::test_in_class`
- `test_file.py::TestClass`
- `test_file.py`
- `foo`

The `test_outside_class` test will be available under the following names:

- `test_file.py::test_outside_class`
- `test_file.py`
- `foo`

The `test_without_name` test will be available under the following names:

- `test_file.py::test_without_name`
- `test_file.py`

Note how some names apply to multiple tests. Depending on `foo` in this case would mean depending on both
`test_in_class` and `test_outside_class`, and depending on `test_file.py` would mean depending on all 3 tests in this
file.

Another example, with parametrization. We'll call this one `test_params.py`:

``` python
@pytest.mark.depends(name='bar')
@pytest.mark.parametrize('num', [
    pytest.param(1, marks=pytest.mark.depends(name='baz')),
    2,
])
def test_with_params(num):
    pass
```

The first run of the test, with `num = 1`, will be available under the following names:

- `test_params.py::test_with_params[num0]`
- `test_params.py::test_with_params`
- `test_params.py`
- `baz`

The second run of the test, with `num = 2`, will be available under the following names:

- `test_params.py::test_with_params[num1]`
- `test_params.py::test_with_params`
- `test_params.py`
- `bar`

Note that that version that got its own mark with `pytest.depends` doesn't have `bar` as name. This is because this
completely replaces the `depends` mark for this test. If you want it to also have `bar` as name, do the following
instead:

``` python
pytest.param(1, marks=pytest.mark.depends(name=['bar', 'baz']))
```

Also note that the first name has a partially autogenerated name. If you want to depend on a single instance of a
parametrized test, it's recommended to use the `pytest.depends` syntax to give it a name rather than depending on the
autogenerated one.