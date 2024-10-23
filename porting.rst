============
Porting tips
============

.. highlight:: python

This section highlights some of the known incompatible changes made
in Python that could break Python scripts and modules that used to work
in prior versions.  The sections are split into retroactive changes made
to all Python releases, and information specific to every Python branch
(compared to the previous one).

This guide is by no means considered complete.  If you can think
of other problems you've hit while porting your packages, please let me
know and I will update it.


Retroactive changes
===================

bpo43882_: urlsplit now strips LF, CR and HT characters
-------------------------------------------------------
Changed in: 2.7.18_p9, 3.6.13_p3, 3.7.10_p3, 3.8.9_p2, 3.9.4_p1

Historically, various urllib.parse_ methods have passed special
characters such as LF, CR and HT through into the split URL components.
This could have resulted in various exploits if Python programs did not
validate the resulting components and used them verbatim.

bpo43882_ attempted to address the issue by making urllib.parse_ strip
the three aforementioned characters from the output of its functions.
This fixed one class of potential issues but at the same time opened
another can of worms.  For example, URL validators that used to check
for dangerous special characters in the split URL components stopped
working correctly.  In the best case, the URL were now sanitized instead
of being rejected.  In the worst, the original unparsed URL with
dangerous characters started being passed through.  See e.g. `django
PR#14349`_ for an example of impact and a fix.

Behavior before::

    >>> urllib.parse.urlparse('https://example.com/bad\nurl')
    ParseResult(scheme='https', netloc='example.com', path='/bad\nurl', params='', query='', fragment='')

Behavior after::

    >>> urllib.parse.urlparse('https://example.com/bad\nurl')
    ParseResult(scheme='https', netloc='example.com', path='/badurl', params='', query='', fragment='')


.. _bpo43882: https://bugs.python.org/issue43882
.. _urllib.parse: https://docs.python.org/3/library/urllib.parse.html
.. _django PR#14349: https://github.com/django/django/pull/14349


.. index:: freethreading

Freethreading CPython versions
==============================
CPython is used the so-called "global interpreter lock" to prevent
multiple threads from executing Python code simultaneously.  It was
designed like this to simplify implementation, but at the cost of
limiting the possible performance gain from use of multiple threads.
Historically, only compiled extensions could temporarily disable it
to take advantage of threading.

Starting with CPython 3.13, a support for an experimental
"freethreading_" mode has been added.  If CPython is built in this
variant, it uses more fine-grained locking mechanisms to make it
possible to run multiple Python threads simultaneously.  Note that this
mode can only improve performance of multithreaded applications,
and (at least at this point) penalizes single-threaded applications.

The freethreaded versions of CPython are not ABI-compatible with
the regular builds.  Therefore, they are slotted as separate versions
with a ``t`` suffix, e.g. Python 3.13 freethreading can be found
as ``dev-lang/python:3.13t``.  They also need to be tested and added
to ``PYTHON_COMPAT`` separately, e.g. as ``python3_13t``.

Note that compiled extensions need to declare support for freethreading
explicitly (see: `C API Extension Support for Free Threading`_).  While
extensions without explicit support can be compiled for freethreading
Python versions, importing them will cause GIL to be reenabled
and therefore defeat the purpose of freethreading build.  To determine
whether a C extension supports freethreading mode, grep the code
for ``Py_MOD_GIL_NOT_USED``.  CPython will also verbosely warn upon
importing extensions without this support.

In general, do not add ``python3_13t`` to ``PYTHON_COMPAT`` in leaf
packages, unless they make use of multithreading and have real gain
from freethreaded versions — otherwise, it may actually be slower than
the regular variant.

For dependency packages, add ``python3_13t`` only after explicitly
testing that the package in question works.  Do not add it
if the package in question installs extensions that do not support
freethreading.  This would penalize the setup, prevent proper testing
and therefore defeat the purpose of separately specifying this target.
Preferably, wait until they do.  For some common dependencies, adding it
may be acceptable, provided that the extensions are optional and that
they are not built for freethreading targets.


.. _freethreading:
   https://docs.python.org/3/howto/free-threading-python.html
.. _C API Extension Support for Free Threading:
   https://docs.python.org/3/howto/free-threading-extensions.html


Python 3.13
===========

See also: `what's new in Python 3.13`_

.. _what's new in Python 3.13:
   https://docs.python.org/3.13/whatsnew/3.13.html


.. index:: cgi

cgi module removal
------------------
Python 3.13 removed the deprecated cgi_ module that provided a number
of utilities for CGI scripts.  The standard library documentation
(as of Python 3.12) provides detailed information on replacing
the deprecated functions.

The said documentation recommends the multipart_ package
as a replacement for some of the functions.  In its context,
the multipart vs. python-multipart packages section of :doc:`migration`
should be consulted as well.

.. _cgi: https://docs.python.org/3.12/library/cgi.html
.. _multipart: https://pypi.org/project/multipart/


Docstring dedenting
-------------------
Prior to Python 3.13, all whitespace in docstrings would be preserved
and exposed in the ``__doc__`` attribute.  Starting with 3.13,
docstrings are instead stripped and dedented.  For example, consider
the following docstring::

    def frobnicate(thing):
        """Frobnicate the thing

        Do some magic frobnication on the thing.

        Example::

             frobnicated_thing = frobnicate(thing)
        """

In Python 3.12, all whitespace is preserved in ``__doc__``, yielding:

.. code-block:: text

    Frobnicate the thing

        Do some magic frobnication on the thing.

        Example::

             frobnicated_thing = frobnicate(thing)

Python 3.13 instead strips leading whitespace from the first line,
and the common amount of whitespace from the subsequent lines, yielding:

.. code-block:: text

    Frobnicate the thing

    Do some magic frobnication on the thing.

    Example::

         frobnicated_thing = frobnicate(thing)

This can break some tests that rely on specific ``__doc__`` values.
To ensure consistent results, `inspect.cleandoc()`_ can be used
to perform the same operation in older Python versions, i.e.::

    assert inspect.cleandoc(frobnicate.__doc__) == expected_doc


.. _inspect.cleandoc():
   https://docs.python.org/3.13/library/inspect.html#inspect.cleandoc


Python 3.12
===========

See also: `what's new in Python 3.12`_

.. _what's new in Python 3.12:
   https://docs.python.org/3.12/whatsnew/3.12.html


.called_with (and other invalid assertions) now trigger an error
----------------------------------------------------------------
It is not uncommon for test suites to write invalid assertions such as::

    with unittest.mock.patch("...") as foo_mock:
        ...

    assert foo_mock.called_with(...)

Prior to Python 3.12, such assertions would silently pass.  Since
the ``.called_with()`` method does not exist, a ``MagicMock`` object
is returned and it evaluates to ``True`` in boolean context.

Starting with Python 3.12, an exception is raised instead::

    AttributeError: 'called_with' is not a valid assertion. Use a spec for the mock if 'called_with' is meant to be an attribute.

The fix is to use the correct ``.assert_called_with()`` method
or similar::

    with unittest.mock.patch("...") as foo_mock:
        ...

    foo_mock.assert_called_with(...)

See the unittest.mock_ documentation for the complete list of available
assertions.

Please note that since the original code did not actually test anything,
fixing the test case may reveal failed expectations.


.. _unittest.mock: https://docs.python.org/3.12/library/unittest.mock.html


Deprecated test method alias removal
------------------------------------
Python 3.12 removes multiple deprecated test method aliases, such
as ``assertEquals()`` and ``assertRegexpMatches()``.  The documentation
provides `a list of removed aliases and their modern replacements`_.

It should be noted that all of the new methods are available since
Python 3.2 (and most even earlier), so the calls can be replaced without
worrying about backwards compatibility.

Most of the time, it should be possible to trivially ``sed`` the methods
in ebuild without having to carry a patch, e.g.:

.. code-block:: bash

    src_prepare() {
        # https://github.com/byroot/pysrt/commit/93f52f6d4f70f4e18dc71deeaae0ec1e9100a50f
        sed -i -e 's:assertEquals:assertEqual:' tests/*.py || die
        distutils-r1_src_prepare
    }


.. _a list of removed aliases and their modern replacements:
   https://docs.python.org/3.12/whatsnew/3.12.html#id3


Python 3.11
===========

See also: `what's new in Python 3.11`_

.. _what's new in Python 3.11:
   https://docs.python.org/3.11/whatsnew/3.11.html


Generator-based coroutine removal (asyncio.coroutine)
-----------------------------------------------------
Support for `generator-based coroutines`_ has been deprecated since
Python 3.8, and is finally removed in 3.11.  This usually results
in the following error::

    AttributeError: module 'asyncio' has no attribute 'coroutine'

The recommended solution is to use `PEP 492 coroutines`_.  They are
available since Python 3.5.  This means replacing
the ``@asyncio.coroutine`` decorator with ``async def`` keyword,
and ``yield from`` with ``await``.

For example, the following snippet::

    @asyncio.coroutine
    def foo():
        yield from asyncio.sleep(5)

would become::

    async def foo():
        await asyncio.sleep(5)


.. _generator-based coroutines:
   https://docs.python.org/3.10/library/asyncio-task.html#generator-based-coroutines
.. _PEP 492 coroutines:
   https://docs.python.org/3.10/library/asyncio-task.html#coroutines


inspect.getargspec() and inspect.formatargspec() removal
--------------------------------------------------------
The `inspect.getargspec()`_ (deprecated since Python 3.0)
and `inspect.formatargspec()`_ (deprecated since Python 3.5) functions
are both removed in Python 3.11.

The `inspect.getargspec()`_ function provides a legacy interface
to inspect the signature of callables.  It is replaced
by the object-oriented `inspect.signature()`_ API (available since
Python 3.3), or a mostly compatible `inspect.getfullargspec()`_ function
(available since Python 3.0).

For example, a trivial function would yield the following results::

    >>> def foo(p1, p2, /, kp3, kp4 = 10, kp5 = None, *args, **kwargs):
    ...     pass
    ...
    >>> inspect.getargspec(foo)
    ArgSpec(args=['p1', 'p2', 'kp3', 'kp4', 'kp5'],
            varargs='args',
            keywords='kwargs',
            defaults=(10, None))
    >>> inspect.getfullargspec(foo)
    FullArgSpec(args=['p1', 'p2', 'kp3', 'kp4', 'kp5'],
                varargs='args',
                varkw='kwargs',
                defaults=(10, None),
                kwonlyargs=[],
                kwonlydefaults=None,
                annotations={})
    >>> inspect.signature(foo)
    <Signature (p1, p2, /, kp3, kp4=10, kp5=None, *args, **kwargs)>

The named tuple returned by `inspect.getfullargspec()`_ starts with
the same information, except that the key used to hold the name
of ``**`` parameter is ``varkw`` rather than ``keywords``.
`inspect.signature()`_ returns a ``Signature`` object.

Both of the newer functions support keyword-only arguments and type
annotations::

    >>> def foo(p1: int, p2: str, /, kp3: str, kp4: int = 10,
    ...         kp5: float = None, *args, k6: str, k7: int = 12,
    ...         k8: float, **kwargs) -> float:
    ...     pass
    ...
    >>> inspect.getfullargspec(foo)
    FullArgSpec(args=['p1', 'p2', 'kp3', 'kp4', 'kp5'],
                varargs='args',
                varkw='kwargs',
                defaults=(10, None),
                kwonlyargs=['k6', 'k7', 'k8'],
                kwonlydefaults={'k7': 12},
                annotations={'return': <class 'float'>,
                             'p1': <class 'int'>,
                             'p2': <class 'str'>,
                             'kp3': <class 'str'>,
                             'kp4': <class 'int'>,
                             'kp5': <class 'float'>,
                             'k6': <class 'str'>,
                             'k7': <class 'int'>,
                             'k8': <class 'float'>})
    >>> inspect.signature(foo)
    <Signature (p1: int, p2: str, /, kp3: str, kp4: int = 10,
                kp5: float = None, *args, k6: str, k7: int = 12,
                k8: float, **kwargs) -> float>

One notable difference between `inspect.signature()`_ and the two other
functions is that the latter always include the 'self' argument
of method prototypes, while the former skips it if the method is bound
to an object.  That is::

    >>> class foo:
    ...     def x(self, bar):
    ...         pass
    ...
    >>> inspect.getargspec(foo.x)
    ArgSpec(args=['self', 'bar'], varargs=None, keywords=None, defaults=None)
    >>> inspect.getargspec(foo().x)
    ArgSpec(args=['self', 'bar'], varargs=None, keywords=None, defaults=None)
    >>> inspect.signature(foo.x)
    <Signature (self, bar)>
    >>> inspect.signature(foo().x)
    <Signature (bar)>

The `inspect.formatargspec()`_ function provides a pretty-formatted
argument spec from the tuple returned by `inspect.getfullargspec()`_
(or `inspect.getargspec()`_).  It is replaced by stringification
of ``Signature`` objects::

    >>> def foo(p1: int, p2: str, /, kp3: str, kp4: int = 10,
    ...         kp5: float = None, *args, k6: str, k7: int = 12,
    ...         k8: float, **kwargs) -> float:
    ...     pass
    ...
    >>> inspect.formatargspec(*inspect.getfullargspec(foo))
    '(p1: int, p2: str, kp3: str, kp4: int=10, kp5: float=None, '
    '*args, k6: str, k7: int=12, k8: float, **kwargs) -> float'
    >>> str(inspect.signature(foo))
    '(p1: int, p2: str, /, kp3: str, kp4: int = 10, kp5: float = None, '
    '*args, k6: str, k7: int = 12, k8: float, **kwargs) -> float'


.. _inspect.getargspec():
   https://docs.python.org/3.10/library/inspect.html#inspect.getargspec
.. _inspect.formatargspec():
   https://docs.python.org/3.10/library/inspect.html#inspect.formatargspec
.. _inspect.getfullargspec():
   https://docs.python.org/3.10/library/inspect.html#inspect.getfullargspec
.. _inspect.signature():
   https://docs.python.org/3.10/library/inspect.html#inspect.signature


Python 3.10
===========

See also: `what's new in Python 3.10`_

.. _what's new in Python 3.10:
   https://docs.python.org/3/whatsnew/3.10.html


configure: No package 'python-3.1' found
----------------------------------------
automake prior to 1.16.3 wrongly recognized Python 3.10 as 3.1.
As a result, build with Python 3.10 fails:

.. code-block:: console

    checking for python version... 3.1
    checking for python platform... linux
    checking for python script directory... ${prefix}/lib/python3.10/site-packages
    checking for python extension module directory... ${exec_prefix}/lib/python3.10/site-packages
    checking for PYTHON... no
    configure: error: Package requirements (python-3.1) were not met:

    No package 'python-3.1' found

    Consider adjusting the PKG_CONFIG_PATH environment variable if you
    installed software in a non-standard prefix.

    Alternatively, you may set the environment variables PYTHON_CFLAGS
    and PYTHON_LIBS to avoid the need to call pkg-config.
    See the pkg-config man page for more details.
    Error: Process completed with exit code 1.

To resolve this in ebuild, you need to autoreconf with the Gentoo
distribution of automake::

    inherit autotools

    # ...

    src_prepare() {
        default
        eautoreconf
    }

The upstream fix is to create new distfiles using automake-1.16.3+.


distutils.sysconfig deprecation
-------------------------------
Upstream intends to remove distutils by Python 3.12.  Python 3.10 starts
throwing deprecation warnings for various distutils modules.
The distutils.sysconfig is usually easy to port.

The following table summarizes replacements for common path getters.

  =================================== ==================================
  distutils.sysconfig call            sysconfig replacement
  =================================== ==================================
  ``get_python_inc(False)``           ``get_path("include")``
  ``get_python_inc(True)``            ``get_path("platinclude")``
  ``get_python_lib(False, False)``    ``get_path("purelib")``
  ``get_python_lib(True, False)``     ``get_path("platlib")``
  ``get_python_lib(False, True)``     ``get_path("stdlib")``
  ``get_python_lib(True, True)``      ``get_path("platstdlib")``
  =================================== ==================================

For both functions, omitted parameters default to ``False``.  There is
no trivial replacement for the variants with ``prefix`` argument.


Python 3.9
==========

See also: `what's new in Python 3.9`_

.. _what's new in Python 3.9:
   https://docs.python.org/3/whatsnew/3.9.html


base64.encodestring / base64.decodestring removal
-------------------------------------------------
Python 3.9 removes the deprecated ``base64.encodestring()``
and ``base64.decodestring()`` functions.  While they were deprecated
since Python 3.1, many packages still use them today.

The drop-in Python 3.1+ replacements are ``base64.encodebytes()``
and ``base64.decodebytes()``.  Note that contrary to the names, the old
functions were simply aliases to the byte variants in Python 3
and *required* the arguments to be ``bytes`` anyway.

If compatibility with Python 2 is still desired, then the byte variants
ought to be called on 3.1+ and string variants before that.  The old
variants accept both byte and unicode strings on Python 2.

Example compatibility import::

    import sys

    if sys.version_info >= (3, 1):
        from base64 import encodebytes as b64_encodebytes
    else:
        from base64 import encodestring as b64_encodebytes

Note that the ``base64`` module also provides ``b64encode()``
and ``b64decode()`` functions that were not renamed.  ``b64decode()``
can be used as a drop-in replacement for ``decodebytes()``.  However,
``b64encode()`` does not insert newlines to split the output
like ``encodebytes()`` does, and instead returns a single line
of base64-encoded data for any length of output.


Python 3.8
==========

See also: `what's new in Python 3.8`_

.. _what's new in Python 3.8:
   https://docs.python.org/3/whatsnew/3.8.html


python-config and pkg-config no longer list Python library by default
---------------------------------------------------------------------
Until Python 3.7, the ``python-X.Y`` pkg-config file and python-config
tool listed the Python library.  Starting with 3.8, this is no longer
the case.  If you are building Python extensions, this is fine (they
are not supposed to link directly to libpython).

If you are building programs that need to embed the Python interpreter,
new ``python-X.Y-embed`` pkg-config file and ``--embed`` parameter
are provided for the purpose.

.. code-block:: console

    $ pkg-config --libs python-3.7
    -lpython3.7m
    $ pkg-config --libs python-3.8

    $ pkg-config --libs python-3.8-embed
    -lpython3.8

To achieve backwards compatibility, you should query
``python-X.Y-embed`` first and fall back to ``python-X.Y``.


Replacing the toml package
==========================

The old toml_ package is no longer maintained.  It was last released
in November 2020 and it was never updated to implement TOML 1.0.
The recommended alternatives are:

- the built-in tomllib_ module (since Python 3.11) with fallback to
  tomli_ package for reading TOML files

- the tomli-w_ package for writing TOML files

- the tomlkit_ package for editing already existing TOML files
  while preserving style


Porting to tomllib/tomli without toml fallback
----------------------------------------------
Using a combination of tomllib_ and tomli_ is the recommended approach
for packages that only read TOML files, or both read and write them
but do not need to preserve style.  The tomllib module is available
since Python 3.11, while tomli versions providing a compatible API
are compatible with Python 3.6 and newer.

The key differences between toml_ and tomllib/tomli are:

- the ``load()`` function accepts only a file object open for reading
  in binary mode whereas toml expects a path or a file object open
  for reading in text mode

- the exception raised for invalid input is named ``TOMLDecodeError``
  where it is named ``TomlDecodeError`` in toml

For example, the following code::

    import toml

    try:
        d1 = toml.load("in1.toml")
    except toml.TomlDecodeError:
        d1 = None

    with open("in2.toml", "r") as f:
        d2 = toml.load(f)

    d3 = toml.loads('test = "foo"\n')

would normally be written as::

    import sys

    if sys.version_info >= (3, 11):
        import tomllib
    else:
        import tomli as tomllib

    try:
        # tomllib does not accept paths
        with open("in1.toml", "rb") as f:
            d1 = tomllib.load(f)
    # the exception uses uppercase "TOML"
    except tomllib.TOMLDecodeError:
        d1 = None

    # the file must be open in binary mode
    with open("in2.toml", "rb") as f:
        d2 = tomllib.load(f)

    d3 = tomllib.loads('test = "foo"\n')

The following dependency string:

.. code-block:: toml

    dependencies = [
        "toml",
    ]

would be replaced by:

.. code-block:: toml

    dependencies = [
        "tomli >= 1.2.3; python_version < '3.11'",
    ]


Porting to tomllib/tomli with toml fallback
-------------------------------------------
If upstream insists on preserving compatibility with EOL versions
of Python, it is possible to use a combination of tomllib_, tomli_
and toml_.  Unfortunately, the incompatibilites in API need to be taken
into consideration.

For example, a backwards compatible code for loading a TOML file could
look like the following::

    import sys

    try:
        if sys.version_info >= (3, 11):
            import tomllib
        else:
            import tomli as tomllib

        try:
            with open("in1.toml", "rb") as f:
                d1 = tomllib.load(f)
        except tomllib.TOMLDecodeError:
            d1 = None
    except ImportError:
        import toml

        try:
            with open("in1.toml", "r") as f:
                d1 = toml.load(f)
        except toml.TomlDecodeError:
            d1 = None

In this case, the dependency string becomes more complex:

.. code-block:: toml

    dependencies = [
        "tomli >= 1.2.3; python_version >= '3.6' and python_version < '3.11'",
        "toml; python_version < '3.6'",
    ]


Porting to tomli-w
------------------
tomli-w_ provides a minimal module for dumping TOML files.

The key differences between toml_ and tomli-w are:

- the ``dump()`` function takes a file object open for writing in binary
  mode whereas toml expected a file object open for writing in text mode

- providing a custom encoder instance is not supported

For example, the following code::

    import toml

    with open("out.toml", "w") as f:
        toml.dump({"test": "data"}, f)

would be replaced by::

    import tomli_w

    with open("out.toml", "wb") as f:
        tomli_w.dump({"test": "data"}, f)

Note that when both reading and writing TOML files is necessary, two
modules need to be imported and used separately rather than one.


.. _toml: https://pypi.org/project/toml/
.. _tomllib: https://docs.python.org/3.11/library/tomllib.html
.. _tomli: https://pypi.org/project/tomli/
.. _tomli-w: https://pypi.org/project/tomli-w/
.. _tomlkit: https://pypi.org/project/tomlkit/
