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
