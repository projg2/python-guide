============================
distutils-r1 legacy concepts
============================

.. highlight:: bash

This section describes concepts specific to the legacy mode
of the ``distutils-r1`` eclass.  When using the modern PEP 517 mode,
none of the features described here are available.


.. index:: DISTUTILS_USE_SETUPTOOLS

Different build system variations
=================================
The commonly used build systems specific to Python packages can be
classified for eclass support into following groups:

1. Plain distutils (built-in in Python).
2. Setuptools and its direct derivatives (e.g. pbr).
3. ``pyproject.toml``-based build systems (Flit, Poetry).

The eclass supports the first two directly.  Support for Flit and Poetry
is provided through the ``dev-python/pyproject2setuppy`` package that
converts the package's metadata to setuptools call.

In addition to being a build system, setuptools provides runtime
facilities via the ``pkg_resources`` module.  If these facilities
are used, the package needs to have a runtime dependency
on ``dev-python/setuptools``.  Otherwise, a build-time dependency
is sufficient.


DISTUTILS_USE_SETUPTOOLS
------------------------
The most common case right now is a package using setuptools as a build
system, and therefore needing a build-time dependency only.  This
is the eclass' default.  If your package does not fit this profile,
you can set ``DISTUTILS_USE_SETUPTOOLS`` variable to one
of the supported values:

- ``no`` — pure distutils use (no extra dependencies).
- ``bdepend`` — build-time use of setuptools (``BDEPEND``
  on ``dev-python/setuptools``).
- ``rdepend`` — build- and runtime use of setuptools (``BDEPEND``
  and ``RDEPEND`` on ``dev-python/setuptools``).
- ``pyproject.toml`` — use of Flit or Poetry (``BDEPEND``
  on ``dev-python/pyproject2toml`` and ``dev-python/setuptools``).
- ``manual`` — special value indicating that the eclass logic misbehaves
  and the dependency needs to be specified manually.  Strongly
  discouraged, please report a bug and/or fix the package instead.

The cases for particular values are explained in subsequent sections.

The Gentoo repository includes a post-install QA check that verifies
whether the value of ``DISTUTILS_USE_SETUPTOOLS`` is correct,
and reports if it is most likely incorrect.  This is why it is important
to use the variable rather than specifying the dependency directly.
An example report is::

     * DISTUTILS_USE_SETUPTOOLS value is probably incorrect
     *   have:     DISTUTILS_USE_SETUPTOOLS=bdepend (or unset)
     *   expected: DISTUTILS_USE_SETUPTOOLS=rdepend

The value needs to be set before inheriting the eclass:

.. code-block:: bash
   :emphasize-lines: 7

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{10..13} pypy3 )
    DISTUTILS_USE_SETUPTOOLS=rdepend

    inherit distutils-r1

    DESCRIPTION="A configurable sidebar-enabled Sphinx theme"
    HOMEPAGE="https://github.com/bitprophet/alabaster"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="BSD"
    KEYWORDS="~alpha ~amd64 ~arm ~arm64 ~hppa ~ia64 ~m68k ~mips ~ppc ~ppc64 ~s390 ~sh ~sparc ~x86 ~x64-solaris"
    SLOT="0"


distutils and setuptools build systems
--------------------------------------
Distutils and setuptools are the two most common build systems
for Python packages right now.  Their common feature is that they use
a ``setup.py`` script that interfaces with the build system.  Generally,
you can determine which of the two build systems are being used
by looking at the imports in ``setup.py``, in particular from which
module the ``setup`` function is imported.

Distutils-based packages (``DISTUTILS_USE_SETUPTOOLS=no``) use e.g.:

.. code-block:: python

    from distutils import setup

Setuptools-based package (``DISTUTILS_USE_SETUPTOOLS=bdepend``, unset
or possibly ``rdepend`` as indicated by the subsequent sections) use:

.. code-block:: python

    from setuptools import setup

In some cases, upstreams find it convenient to alternatively support
both setuptools and distutils.  A commonly used snippet is:

.. code-block:: python

    try:
        from setuptools import setup
    except ImportError:
        from distutils import setup

However, non-fixed build system choice can be problematic to Gentoo
users.  This is because pure distutils installs egg-info data as a
single file, while setuptools install the same data as a directory
(using the same path).  Therefore, if you rebuild the same version
of the package with a different build system than before, you end up
trying to replace a file with a directory or the other way around.
This is not permitted by the PMS and not handled cleanly by the package
managers.

You must always ensure that a single build system will be used
unconditionally.  In the case of the condition presented above, it is
sufficient to leave ``DISTUTILS_USE_SETUPTOOLS`` at its default value
as that will ensure that setuptools is installed and therefore
the fallback will never take place.  However, patching ``setup.py`` may
be necessary if you want to force distutils (e.g. to enable clean
bootstrap) or the upstream condition requiers that.


Setuptools' entry points
------------------------
.. Important::

   With removal of Python 3.7, the correct ``DISTUTILS_USE_SETUPTOOLS``
   value for packages using entry points changed to ``bdepend``.

*Entry points* provide the ability to expose some of the package's
Python functions to other packages.  They are commonly used to implement
plugin systems and by setuptools themselves to implement wrapper scripts
for starting programs.

Entry points are defined as ``entry_points`` argument to the ``setup()``
function, or ``entry_points`` section in ``setup.cfg``.  They are
installed in the package's egg-info as ``entry_points.txt``.  In both
cases, they are grouped by entry point type, and defined as a dictionary
mapping entry points names to the relevant functions.

For our purposes, we are only interested in entry points used to define
wrapper scripts, the ``console_scripts`` and ``gui_scripts`` groups,
as they are installed with the package itself.  As for plugin systems,
it is reasonable to assume that the installed plugins are only
meaningful to the package using them, and therefore that the package
using them will depend on the appropriate metadata provider.

Old versions of setuptools used to implement the script wrappers using
``pkg_resources`` package.  Modern versions of setuptools use
the following logic:

1. If ``importlib.metadata`` module is available (Python 3.8+), use it.
   In this case, no external dependencies are necessary.

2. If ``importlib_metadata`` backport is available, use it.  It is
   provided by ``dev-python/importlib_metadata``.

3. Otherwise, fall back to ``pkg_resources``.  It is provided
   by ``dev-python/setuptools``.

Since Python 3.7 is no longer present in Gentoo, new ebuilds do not
need any additional dependencies for entry points and should use
the default value (i.e. remove ``DISTUTILS_USE_SETUPTOOLS``).

For the time being, the QA check for incorrect values is accepting
both the new value and the old ``rdepend`` value.  If you wish to be
reminded about the update, you can add the following variable to your
``make.conf``::

    DISTUTILS_STRICT_ENTRY_POINTS=1

Please note that in some cases ``rdepend`` can still be the correct
value, if there are `other runtime uses of setuptools`_.  In some cases
the QA check will also trigger the wrong value because of leftover
explicit dependencies on setuptools.


Other runtime uses of setuptools
--------------------------------
Besides the generated wrapper scripts, the package code itself may use
the ``setuptools`` or ``pkg_resources`` packages.  The common cases
for this include getting package metadata and resource files.  This
could also be a case for plugin managers and derived build systems.

As a rule of thumb, if any installed Python file imports ``setuptools``
or ``pkg_resources``, the package needs to use the value of ``rdepend``.

The QA check determines that this is the case by looking at the upstream
dependencies (``install_requires``) installed by the package.  It is
quite common for packages to miss the dependency, or have a leftover
dependency.  If ``install_requires`` does not match actual imports
in the installed modules, please submit a patch upstream.


pyproject.toml-based projects
-----------------------------
The newer build systems used for Python packages avoid supplying
``setup.py`` and instead declare package's metadata and build system
information in ``pyproject.toml``.  Examples of these build systems
are Flit and Poetry.

These build systems are generally very heavy and do not support plain
installation to a directory.  For this reason, Gentoo is using
``dev-python/pyproject2setuppy`` to provide a thin wrapper for
installing these packages using setuptools.

To enable the necessary eclass logic and add appropriate build-time
dependencies, specify the value of ``pyproject.toml``
to ``DISTUTILS_USE_SETUPTOOLS``.

Strictly speaking, both Flit and Poetry do support entry points,
and therefore some packages actually need a runtime dependency
on setuptools.  This is a known limitation, and it will probably
not be addressed for the same reason as the logic for setuptools' entry
points is not updated.


.. index:: DISTUTILS_IN_SOURCE_BUILD

In-source vs out-of-source builds
=================================
In the general definition, an *out-of-source build* is a build where
output files are placed in a directory separate from source files.
By default, distutils and its derivatives always do out-of-source builds
and place output files in subdirectories of ``build`` directory.

Conversely, an *in-source build* happens when the output files are
interspersed with source files.  The closest distutils equivalent
of an in-source build is the ``--inplace`` option of ``build_ext``
that places compiled C extensions alongside Python module sources.

``distutils-r1`` shifts this concept a little.  When performing
an out-of-source build (the default), it creates a dedicated output
directory for every Python interpreter enabled, and then uses it
throughout all build and install steps.

It should be noted that unlike build systems such as autotools or CMake,
out-of-source builds in distutils are not executed from the build
directory.  Instead, the setup script is executed from source directory
and passed path to build directory.

Sometimes out-of-source builds are incompatible with custom hacks used
upstream.  This could be a case if the setup script is writing
implementation-specific changes to the source files (e.g. using ``2to3``
to convert them to Python 3) or relying on specific build paths.
For better compatibility with those cases, the eclass provides
an in-source build mode enabled via ``DISTUTILS_IN_SOURCE_BUILD``.

In this mode, the eclass creates a separate copy of the source directory
for each Python implementation, and then runs the build and install
steps inside that copy.  As a result, any changes done to the source
files are contained within the copy used for the current interpreter.

.. code-block:: bash
   :emphasize-lines: 23

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    DISTUTILS_USE_SETUPTOOLS=no
    PYTHON_COMPAT=( python3_{10..13} pypy3 )
    PYTHON_REQ_USE="xml(+)"

    inherit distutils-r1 pypi

    DESCRIPTION="Collection of extensions to Distutils"
    HOMEPAGE="
        https://github.com/pypa/setuptools
        https://pypi.org/project/setuptools/
    "

    LICENSE="MIT"
    SLOT="0"
    KEYWORDS="~alpha ~amd64 ~arm ~arm64 ~hppa ~ia64 ~m68k ~mips ~ppc ~ppc64 ~riscv ~s390 ~sh ~sparc ~x86 ~x64-cygwin ~amd64-linux ~x86-linux ~ppc-macos ~x64-macos ~x86-macos ~sparc-solaris ~sparc64-solaris ~x64-solaris ~x86-solaris"

    # Force in-source build because build system modifies sources.
    DISTUTILS_IN_SOURCE_BUILD=1


.. index:: distutils_install_for_testing

Installing the package before testing
=====================================
The tests are executed in ``src_test`` phase, after ``src_compile``
installed package files into the build directory.  The eclass
automatically adds appropriate ``PYTHONPATH`` so that the installed
Python modules and extensions are used during testing.  This works
for the majority of packages.

However, some test suites will not work correctly unless the package
has been properly installed via ``setup.py install``.  This may apply
specifically to packages calling their executables that are created
via entry points, various plugin systems or the use of package metadata.

The ``distutils_install_for_testing`` function runs ``setup.py install``
into a temporary directory, and adds the appropriate paths to ``PATH``
and ``PYTHONPATH``.

This function currently supports two install layouts:

- the standard *root directory* layout that is enabled
  via ``--via-root``,

- a virtualenv-alike *venv* layout that is enabled via ``--via-venv``.


The eclass defaults to the root directory layout that is consistent
with the layout used for the actual install.  This ensures that
the package's scripts are found on ``PATH``, and the package metadata
is found via ``importlib.metadata`` / ``pkg_resources``.  It should
be sufficient to resolve the most common test problems.

In some cases, particularly packages that do not preserve ``PYTHONPATH``
correctly, the virtualenv-alike layout (``--via-venv``) is better.
Through wrapping the Python interpreter itself, it guarantees that
the packages installed in the test environment are found independently
of ``PYTHONPATH`` (just like a true venv).  It should cover the few
extreme cases.

In EAPIs prior to 8, an additional legacy ``--via-home`` layout used
to be supported.  It historically used to be necessary to fix problems
with some packages.  However, the underlying issues probably went away
along with old versions of Python, and the `removal of site.py hack`_
has broken it for most of the consumers.

.. code-block:: bash

    python_test() {
        distutils_install_for_testing
        epytest --no-network
    }


.. _removal of site.py hack:
   https://github.com/pypa/setuptools/commit/91213fb2e7eecde9f5d7582de485398f546e7aa8
