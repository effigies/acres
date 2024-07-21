# Acres: Access resources on your terms

[![PyPI release](https://img.shields.io/pypi/v/acres.svg)](https://pypi.python.org/project/acres/)
[![Codecov Coverage report](https://codecov.io/github/nipreps/acres/graph/badge.svg?token=jVfxERJR5k)](https://codecov.io/github/nipreps/acres)

This module provides simple, consistent access to package resources.

```python
from acres import Loader

loader = Loader(somepkg)

res_text = loader.readable('data/some_resource.txt').read_text()
res_bytes = loader.readable('data/another_resource.bin').read_bytes()

with loader.as_path('data') as data_dir:
    # data_dir is a pathlib.Path until the "with" scope closes

resource_path = loader.cached('data/another_resource.bin')
# The path pointed to by resource_path will exist until interpreter exit
```

## Module loader

The primary use case for `acres.Loader` is to allow a directory module to provide access
to its resources.

Suppose you have a module structure:

```
src/
  mypkg/
    data/
      resourceDir/
        ...
      __init__.py
      resource.ext
    __init__.py
    ...
```

In `src/mypkg/data/__init__.py`, add:

```python
'''Data package

.. autofunction:: load_resource

.. automethod:: load_resource.readable

.. automethod:: load_resource.as_path

.. automethod:: load_resource.cached
'''

from acres import Loader

load_resource = Loader(__package__)
```

`mypkg.data.load_resource()` is now a function that will return a `Path` to a
resource that is guaranteed to exist until interpreter exit:

```python
from mypkg.data import load_resource

resource_file: Path = load_resource('resource.ext')
```

For additional control, you can use `load_resource.readable()` to return a `Path`-like
object that implements `.read_text()` and `.read_bytes()`:

```python
resource_contents: bytes = load_resource.readable('resource.ext').read_bytes()
```

Or a context manager with a limited lifetime:

```python
with load_resource.as_path('resourceDir') as resource_dir:
    # Work with the contents of `resource_dir` as a `Path`

# Outside the `with` block, `resource_dir` may no longer point to an existing path.
```

Note that `load_resource()` is a shorthand for `load_resource.cached()`,
whose explicitness might be more to your taste.

## Interpreter-scoped resources, locally scoped loaders

`Loader.cached` uses a global cache. This ensures that cached files do not get
unloaded if a `Loader()` instance is garbage collected, and it also ensures that
instances created cheaply and garbage collected when out of scope.

## Why acres?

`importlib.resources` provides a simple, compsable interface, using the
`files()` and `as_file()` functions:

```python
from importlib.resources import files, as_file

with as_file(files(my_module) / 'data' / 'resource.ext') as resource_path:
    # Interact with resource_path as a pathlib.Path

# resource_path *may* no longer exist
```

`files()` returns a [Traversable][] object, which is similar to a [pathlib.Path][],
except that the object may not actually exist, so `os` functions may not work
correctly on it. `as_files()` turns a `Traversable` into a `Path` that exists on
the filesystem for the duration of the `with` block.

To make matters more complicated, if a package is unpacked on the filesystem,
`files()` returns an actual `Path` object. It is therefore easy to miss bugs
where a true `Path` is always needed but a `Traversable` may be returned when
the package is zipped.

Finally, the scoping of `as_files()` is frequently inconvenient. If you pass a
`Path` to an object inside an `as_files()` context, but the resource access is
deferred, the `Path` may point to a nonexistent file or directory. Again,
this bug will only exhibit when the package is zipped.

`acres.Loader` aims to clearly delineate the scopes and capabilities of
the accessed resources, including providing an interpreter-lifetime scope.

[Traversable]: https://docs.python.org/3/library/importlib.resources.abc.html#importlib.resources.abc.Traversable
[pathlib.Path]: https://docs.python.org/3/library/pathlib.html#pathlib.Path
