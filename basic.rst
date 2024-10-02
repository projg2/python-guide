=============
Common basics
=============

.. highlight:: bash

The various eclasses in python-r1 try to follow a single design.  You
will probably use more than one of them, so it is worthwhile to shortly
explain the common bits used by all of them, as well as the non-obvious
differences between them.


.. index:: PYTHON_COMPAT

PYTHON_COMPAT
=============
The ``PYTHON_COMPAT`` variable is used by all Python eclasses, and must
be declared in all ebuilds before they are inherited.  It specifies
the list of Python implementations supported by the package.

The valid values are:

- ``pythonX_Y`` for CPython X.Y
- ``pypy3`` for PyPy3.

Typical use::

    PYTHON_COMPAT=( python3_{10..13} pypy3 )
    inherit python-single-r1


.. index:: PYTHON_DEPS
.. index:: PYTHON_REQUIRED_USE

PYTHON_DEPS and PYTHON_REQUIRED_USE
===================================
The ``python-any-r1``, ``python-single-r1`` and ``python-r1`` eclasses
all assume that the developer is responsible for explicitly putting
the dependency strings and USE requirements in correct variables.
This is meant to consistently cover packages that use Python
unconditionally and conditionally, at build time and at runtime.

For ``python-single-r1`` and ``python-r1``, the most basic form to use
Python unconditionally is to define the following::

    REQUIRED_USE=${PYTHON_REQUIRED_USE}

    RDEPEND=${PYTHON_DEPS}
    BDEPEND=${RDEPEND}

For ``python-any-r1``, only build-time dependencies are used::

    BDEPEND=${PYTHON_DEPS}

This does not apply to ``distutils-r1`` as it does the above assignment
by default.


.. index:: EPYTHON
.. index:: PYTHON

Python environment
==================
The eclasses commonly use the concept of *Python environment*.  This
means a state of environment enforcing a particular Python
implementation.  Whenever the ebuild code is run inside this
environment, two variables referring to the specific Python interpreter
are being exported:

- ``EPYTHON`` containing the interpreter's basename (also used
  as the implementation identifier), e.g. ``python3.10``
- ``PYTHON`` containing the absolute final path to the interpreter,
  e.g. ``/usr/bin/python3.10``

The full path should only be used to provide the value that should
be embedded in the installed programs, e.g. in the shebangs.
For spawning Python during the build, ``EPYTHON`` is preferable.

.. Warning::

   Using full path rather than the basename will bypass the virtualenv
   created by ``distutils-r1.eclass`` in PEP 517 mode.  This may cause
   failures to import Python modules, or use of the previous installed
   version rather than the just-built one.  Using ``${EPYTHON}``
   resolves these problems.

Wrappers for ``python``, ``pythonN`` and some common tools are provided
in PATH, and ``/usr/bin/python`` etc. also enforce the specific
implementation via python-exec (for programs that hardcode full path).

The environment can be either established in local scope, or globally.
The local scope generally applies to multi-impl packages, and is created
either by calls to ``python_foreach_impl`` from ``python-r1``, or inside
sub-phase functions in ``distutils-r1``.  The global scope setup is done
via calling ``python_setup``, either directly or via default
``pkg_setup`` in ``python-any-r1`` and ``python-single-r1``.


Dependencies in Python packages
===============================
.. Note::

   The following sections focus specifically on dependencies that
   are Python packages.  Python software often depends on external
   tools, libraries written in other programming languages, etc.
   For these dependencies, the usual Gentoo rules apply.


.. index:: BDEPEND
.. index:: DEPEND
.. index:: RDEPEND

The most common dependency types
--------------------------------
The dependencies found in Python packages can usually be classified
into two categories: runtime dependencies and build-time dependencies.

*Runtime dependencies* are packages that are required to be present
in order for the installed Python modules and scripts to be usable.
In general, these are all packages whose modules are imported
in the installed Python files.  Generally runtime dependencies
are not needed at build time and therefore the build systems
do not verify whether they are installed.  However, modern Python
scripts based on entry points often refuse to run if their dependencies
are not satisfied.  Runtime dependencies should be placed
in ``RDEPEND``.

A special subclass of runtime dependencies are *optional runtime
dependencies* (often called 'extra' dependencies).  The dependencies are
optional if the package can still be meaningfully functional when they
are not installed.  This usually means that the package either handles
failing imports gracefully, or that they are imported only in a subset
of package's installed modules and that the package can still be
meaningfully used without importing these modules.

There are multiple approaches to handling optional dependencies.
Depending on the specifics, they can:

1. be added unconditionally to ``RDEPEND`` (if they are considered
   important and/or light enough);

2. be listed as an informational message in ``pkg_postinst`` (usually
   utilizing ``optfeature.eclass``);

3. be added to ``RDEPEND`` conditionally to USE flags (this is only
   acceptable if the package is cheap to rebuild).

*Build-time dependencies* are the packages needed for the package
to be built and installed.  In general, they include the packages
providing the build system.  In some cases, they may also include some
runtime dependencies, e.g. when they are needed to import
the ``__init__.py`` of the package.  As a rule of thumb, if the package
can be built correctly when the specific dependency is not installed,
it does not need to be listed as a build dependency.  Most of the time,
build dependencies belong in ``BDEPEND``.

The ``distutils-r1`` class generally takes care of adding the dependency
on the build system and basic tooling.  However, additional plugins
(e.g. ``dev-python/setuptools_scm``) need to be listed explicitly.

A special class of build-time dependencies are requirements specific
to running the test suite and building documentation.  Most of the time
the former include not only the test runner but also all runtime
dependencies of the package (since the test suite runs its code).
Sometimes this is also required to build documentation.  These classes
of dependencies go into ``BDEPEND`` under ``test`` and ``doc`` USE flags
respectively.

Note that sometimes test dependencies can also be optional (including
optional runtime dependencies).  They should generally be added
unconditionally to ensure maximum test coverage.  Also note that
(as explained further in the Guide), some test dependencies
(e.g. on linters or coverage reporting tools) may actually
be undesirable.

Again, ``distutils-r1`` provides functions to conveniently add support
for common test runner and Sphinx-based documentation.  The former also
takes care of copying ``RDEPEND`` into test dependencies.

Some Python packages include C extensions that depend on external
libraries.  In this case, similarly to non-Python packages,
the dependency on packages providing these libraries needs to go
into ``RDEPEND`` and ``DEPEND`` (not ``BDEPEND``).

Finally, there are Python packages providing C headers such
as ``dev-python/numpy``.  If the package in question uses both headers
and Python code from NumPy, the dependency may need to be included
in all three of ``RDEPEND``, ``DEPEND`` and ``BDEPEND`` (unconditionally
or for tests).


Finding dependency lists from build systems
-------------------------------------------
Most of the modern Python build systems include all the package metadata
in the ``pyproject.toml`` file.  Setuptools are using ``setup.cfg``
and/or ``setup.py``.  Some packages also include custom code to read
dependencies from external files; it is usually worthwhile to look
for ``requirements`` in the name.

.. Warning::

   Unconditional runtime dependencies and unconditional build-time
   dependencies are often enforced by the script wrappers and build
   systems respectively.  If upstream lists spurious dependencies,
   they often need to be explicitly stripped rather than just ommitted
   from ebuild.

The keys commonly used to list specific kinds of dependencies in common
Python build systems:

1. Runtime dependencies (unconditional):

   - `PEP 621`_ metadata: ``project.dependencies``
   - older flit versions: ``tool.flit.metadata.requires``
   - poetry: ``tool.poetry.dependencies`` (note: this also includes
     special ``python`` entry to indicate compatible Python versions)
   - setuptools: ``install_requires``

2. Optional runtime and/or build-time dependencies:

   - `PEP 621`_ metadata: ``project.optional-dependencies``
   - older flit versions: ``tool.flit.metadata.requires-extra``
   - poetry: ``tool.poetry.dependencies`` with ``optional = true``,
     sometimes grouped using ``tool.poetry.extras``
   - setuptools: ``extras_require``

3. Build-time dependencies (unconditional):

   - all ``pyproject.toml`` build systems: ``build-system.requires``
   - poetry: ``tool.poetry.dev-dependencies``
   - setuptools: ``setup_requires`` (deprecated)

4. Test dependencies (in addition to ``RDEPEND``):

   - often listed as ``test`` key in optional dependencies
   - setuptools: ``tests_require`` (deprecated)
   - in some cases they can also be found in ``tox.ini``
     or ``noxfile.py``

5. Doc building dependencies:

   - often listed as ``doc`` key in optional dependencies

6. Python version compatibility:

   - `PEP 621`_ metadata: ``project.requires-python``
   - older flit versions: ``tool.flit.metadata.requires-python``
   - poetry: ``python`` in ``tool.poetry.dependencies``
   - setuptools: ``python_requires``

.. _PEP 621: https://www.python.org/dev/peps/pep-0621
