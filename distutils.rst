============================================
distutils-r1 — standard Python build systems
============================================

.. highlight:: bash

The ``distutils-r1`` eclass is used to facilitate build systems using
``setup.py`` (distutils and its derivatives, notably setuptools)
or ``pyproject.toml`` (flit, poetry).  It is built on top
of ``python-r1`` and ``python-single-r1``, and therefore supports
efficiently building multi-impl and single-impl packages.

Eclass reference: `distutils-r1.eclass(5)`_


Basic use
=========
By default, ``distutils-r1`` sets appropriate metadata variables
and exports a full set of phase functions necessary to install packages
using setuptools.  Therefore, the most simple case of ebuild is:

.. code-block:: bash
   :emphasize-lines: 6,7

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7,8} pypy3  )
    inherit distutils-r1

    DESCRIPTION="Makes working with XML feel like you are working with JSON"
    HOMEPAGE="https://github.com/martinblech/xmltodict/ https://pypi.org/project/xmltodict/"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="MIT"
    SLOT="0"
    KEYWORDS="~amd64 ~arm ~arm64 ~x86"


Dependencies
============
Dependencies on Python packages are declared using the same method
as the underlying eclass — that is, ``python-r1``
or ``python-single-r1``.

In packages using ``dev-python/setuptools``, dependencies are often
specified in ``setup.py`` or ``setup.cfg`` file.
The ``install_requires`` key specifies runtime dependencies,
``setup_requires`` pure build-time dependencies, ``extras_require``
optional dependencies.  Test dependencies are sometimes specified
as one of the 'extras', and sometimes as ``tests_require``.

Setuptools strictly enforces ``setup_requires`` at build time,
and ``tests_require`` when running ``setup.py test``.  Runtime
dependencies are enforced only when starting installed programs
via entry points.

In other cases, dependencies are listed in additional files named
e.g. ``requirements.txt``.  They could be also found in test runner
setup (``tox.ini``) or CI setup files (``.travis.yml``).  Finally, you
can grep source code for ``import`` statements.

In general, you should take special care when listing dependencies
of Python packages.  Upstreams sometimes specify indirect dependencies,
often list packages that are not strictly relevant to Gentoo runs
but used on CI/CD setup, unnecessarily restrict version requirements.

Most of the time, runtime dependencies do not need to be present
at build time.  However, they do need to be copied there if the Python
modules needing them are imported at build time.  Often this is the case
when running tests, hence the following logic is common in Python
ebuilds::

    RDEPEND="..."
    BDEPEND="test? ( ${RDEPEND} )"

There are two different approaches used for optional runtime
dependencies.  Some packages are installing them conditionally to USE
flags (this is generally acceptable as long as package builds quickly),
others list them in ``pkg_postinst()`` messages.  It is recommended
that optional test dependencies are used unconditionally (to ensure
the widest test coverage, and avoid unpredictable test failures on users
who have more dependencies installed).


.. index:: DISTUTILS_SINGLE_IMPL

python-single-r1 variant
========================
Normally, ``distutils-r1`` uses ``python-r1`` to build multi-impl
packages, and this is the recommended mode.  However, in some cases
you will need to use ``python-single-r1`` instead, especially if you
need to depend on other packages using that eclass.

The single-impl mode can be enabled by setting ``DISTUTILS_SINGLE_IMPL``
variable before inheriting the eclass.  The eclass aims to provide
maximum compatibility between these two modes, so most of the existing
code will work with either.  However, the functions specific to
the underlying eclass are not compatible — e.g. the dependencies need
to be rewritten.

.. code-block:: bash
   :emphasize-lines: 8

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6

    PYTHON_COMPAT=( python3_6 )
    PYTHON_REQ_USE="readline"
    DISTUTILS_SINGLE_IMPL=1

    inherit distutils-r1

    DESCRIPTION="Pythonic layer on top of the ROOT framework's PyROOT bindings"
    HOMEPAGE="http://rootpy.org"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="BSD"
    SLOT="0"
    KEYWORDS="~amd64 ~x86 ~amd64-linux ~x86-linux"

    RDEPEND="
        sci-physics/root:=[${PYTHON_SINGLE_USEDEP}]
        dev-python/root_numpy[${PYTHON_SINGLE_USEDEP}]
        $(python_gen_cond_dep '
            dev-python/matplotlib[${PYTHON_USEDEP}]
            dev-python/pytables[${PYTHON_USEDEP}]
            dev-python/termcolor[${PYTHON_USEDEP}]
        ')"

    DEPEND="
        sci-physics/root[${PYTHON_SINGLE_USEDEP}]"


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

    PYTHON_COMPAT=( python2_7 python3_{6,7,8} pypy3 )
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

Since Python 3.7 is no longer present in Gentoo (we are not considering
PyPy3.7 correctness important for the time being), new ebuilds do not
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


.. index:: SETUPTOOLS_SCM_PRETEND_VERSION

setuptools_scm and snapshots
============================
setuptools_scm_ is a package providing additional features for running
inside a VCS checkout, in particular the ability to determine version
from VCS tags.  However, this works correctly only when the package
is built from VCS checkout or an ``sdist`` archive containing
pregenerated metadata.  It does not work when building from a GitHub
snapshot::

    Traceback (most recent call last):
      File "/tmp/executing-0.5.2/setup.py", line 4, in <module>
        setup()
      File "/usr/lib/python3.9/site-packages/setuptools/__init__.py", line 143, in setup
        _install_setup_requires(attrs)
      File "/usr/lib/python3.9/site-packages/setuptools/__init__.py", line 131, in _install_setup_requires
        dist = distutils.core.Distribution(dict(
      File "/usr/lib/python3.9/site-packages/setuptools/dist.py", line 425, in __init__
        _Distribution.__init__(self, {
      File "/usr/lib/python3.9/distutils/dist.py", line 292, in __init__
        self.finalize_options()
      File "/usr/lib/python3.9/site-packages/setuptools/dist.py", line 717, in finalize_options
        ep(self)
      File "/usr/lib/python3.9/site-packages/setuptools_scm/integration.py", line 48, in infer_version
        dist.metadata.version = _get_version(config)
      File "/usr/lib/python3.9/site-packages/setuptools_scm/__init__.py", line 148, in _get_version
        parsed_version = _do_parse(config)
      File "/usr/lib/python3.9/site-packages/setuptools_scm/__init__.py", line 110, in _do_parse
        raise LookupError(
    LookupError: setuptools-scm was unable to detect version for '/tmp/executing-0.5.2'.

    Make sure you're either building from a fully intact git repository or PyPI tarballs. Most other sources (such as GitHub's tarballs, a git checkout without the .git folder) don't contain the necessary metadata and will not work.

    For example, if you're using pip, instead of https://github.com/user/proj/archive/master.zip use git+https://github.com/user/proj.git#egg=proj

This problem can be resolved by providing the correct version externally
via ``SETUPTOOLS_SCM_PRETEND_VERSION``::

    export SETUPTOOLS_SCM_PRETEND_VERSION=${PV}

.. _setuptools_scm: https://pypi.org/project/setuptools-scm/


Parallel build race conditions
==============================
The distutils build system has a major unresolved bug regarding race
conditions.  If the same source file is used to build multiple Python
extensions, the build can start multiple simultaneous compiler processes
using the same output file.  As a result, there is a race between
the compilers writing output file and link editors reading it.  This
generally does not cause immediate build failures but results in broken
extensions causing cryptic issues in reverse dependencies.

For example, a miscompilation of ``dev-python/pandas`` have recently
caused breakage in ``dev-python/dask``::

    /usr/lib/python3.8/site-packages/pandas/__init__.py:29: in <module>
        from pandas._libs import hashtable as _hashtable, lib as _lib, tslib as _tslib
    /usr/lib/python3.8/site-packages/pandas/_libs/__init__.py:13: in <module>
        from pandas._libs.interval import Interval
    pandas/_libs/interval.pyx:1: in init pandas._libs.interval
        ???
    pandas/_libs/hashtable.pyx:1: in init pandas._libs.hashtable
        ???
    pandas/_libs/missing.pyx:1: in init pandas._libs.missing
        ???
    /usr/lib/python3.8/site-packages/pandas/_libs/tslibs/__init__.py:30: in <module>
        from .conversion import OutOfBoundsTimedelta, localize_pydatetime
    E   ImportError: /usr/lib/python3.8/site-packages/pandas/_libs/tslibs/conversion.cpython-38-x86_64-linux-gnu.so: undefined symbol: pandas_datetime_to_datetimestruct

The easiest way to workaround the problem in ebuild is to append ``-j1``
in python_compile_ sub-phase.

The common way of working around the problem upstream is to create
additional .c files that ``#include`` the original file, and use unique
source files for every extension.


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
   :emphasize-lines: 20

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7
    DISTUTILS_USE_SETUPTOOLS=no
    PYTHON_COMPAT=( python3_{6,7,8} pypy3 )
    PYTHON_REQ_USE="xml(+)"

    inherit distutils-r1

    DESCRIPTION="Collection of extensions to Distutils"
    HOMEPAGE="https://github.com/pypa/setuptools https://pypi.org/project/setuptools/"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.zip"

    LICENSE="MIT"
    SLOT="0"
    KEYWORDS="~alpha ~amd64 ~arm ~arm64 ~hppa ~ia64 ~m68k ~mips ~ppc ~ppc64 ~riscv ~s390 ~sh ~sparc ~x86 ~x64-cygwin ~amd64-linux ~x86-linux ~ppc-macos ~x64-macos ~x86-macos ~sparc-solaris ~sparc64-solaris ~x64-solaris ~x86-solaris"

    # Force in-source build because build system modifies sources.
    DISTUTILS_IN_SOURCE_BUILD=1


Sub-phase functions
===================
Ebuilds define phase functions in order to conveniently override parts
of the build process.  ``distutils-r1`` extends this concept
by introducing *sub-phases*.  All ``src_*`` phases in ebuild are split
into two sub-phases: ``python_*`` sub-phases that are run in a loop
for all enabled interpreters, and ``python_*_all`` sub-phases that
comprise the common code to be run only once.

Sub-phase functions behave similarly to phase functions.  They are run
if defined by the ebuild.  If they're not, the default implementation
is run (if any).  The ebuild overrides can call the default
as ``distutils-r1_<sub-phase>``, the same way it can call eclass' phase
function defaults.

There are 10 sub-phases corresponding to 5 phase functions.  They are
run in the following order:

1. ``python_prepare_all`` (for ``src_prepare``, has default)
2. ``python_prepare`` (for each impl.)
3. ``python_configure`` (for ``src_configure``, foreach impl.)
4. ``python_configure_all``
5. ``python_compile`` (for ``src_compile``, for each impl., has default)
6. ``python_compile_all``
7. ``python_test`` (for ``src_test``, for each impl.)
8. ``python_test_all``
9. ``python_install`` (for ``src_install``, for each impl., has default)
10. ``python_install_all`` (has default)

Note that normally all phases are run in the source directory, while
defining ``${BUILD_DIR}`` to a dedicated build directory for each
implementation.  However, if in-source builds are enabled, all phases
are run in these build directories.


.. index:: python_prepare
.. index:: python_prepare_all

python_prepare
--------------

``python_prepare_all`` is responsible for applying changes
to the package sources that are common to all Python implementations.
The default implementation performs the tasks of ``default_src_prepare``
(applying patches), as well as eclass-specific tasks: removing
``ez_setup`` (method of bootstrapping setuptools used in old packages)
and handling ``pyproject.toml``.  In the end, the function copies
sources to build dirs if in-source build is requested.

If additional changes need to be done to the package, either this
sub-phase or ``src_prepare`` in general can be replaced.  However,
you should always call the original implementation from your override.
For example, you could use it to strip extraneous dependencies or broken
tests::

    python_prepare_all() {
        # FIXME
        rm tests/test_pytest_plugin.py || die
        sed -i -e 's:test_testcase_no_app:_&:' tests/test_test_utils.py || die

        # remove pointless dep on pytest-cov
        sed -i -e '/addopts/s/--cov=aiohttp//' pytest.ini || die

        distutils-r1_python_prepare_all
    }

``python_prepare`` is responsible for applying changes specific to one
interpreter.  It has no default implementation.  When defined, in-source
builds are enabled implicitly as sources need to be duplicated to apply
implementation-specific changes.

In the following example, it is used to automatically convert sources
to Python 3.  Naturally, this requires the eclass to keep a separate
copy of the sources that remains compatible with Python 2 and this is
precisely why ``python_prepare`` automatically enables in-source builds.

::

    python_prepare() {
        if python_is_python3; then
            2to3 -n -w --no-diffs *.py || die
        fi
    }


.. index:: python_configure
.. index:: python_configure_all

python_configure
----------------

``python_configure`` and ``python_configure_all`` have no default
functionality.  The former is convenient for running additional
configuration steps if needed by the package, the latter for defining
global environment variables.

::

    python_configure() {
        esetup.py configure $(usex mpi --mpi '')
    }

::

    python_configure_all() {
        mydistutilsargs=(
            --resourcepath=/usr/share
            --no-compress-manpages
        )
    }


.. index:: python_compile
.. index:: python_compile_all

python_compile
--------------

``python_compile`` normally builds the package.  It is sometimes used
to pass additional arguments to the build step.  For example, it can
be used to disable parallel extension builds in packages that are broken
with it::

    python_compile() {
        distutils-r1_python_compile -j1
    }


``python_compile_all``
has no default implementation.  It is convenient for performing
additional common build steps, in particular for building
the documentation (see ``distutils_enable_sphinx``).

::

    python_compile_all() {
        use doc && emake -C docs html
    }


.. index:: python_test
.. index:: python_test_all

python_test
-----------

``python_test`` is responsible for running tests.  It has no default
implementation but you are strongly encouraged to provide one (either
directly or via ``distutils_enable_tests``).  ``python_test_all``
can be used to run additional testing code that is not specific
to Python.

::

    python_test() {
        "${EPYTHON}" TestBitVector/Test.py || die "Tests fail with ${EPYTHON}"
    }


.. index:: python_install
.. index:: python_install_all

python_install
--------------

``python_install`` installs the package's Python part.  It is usually
redefined in order to pass additional ``setup.py`` arguments
or to install additional Python modules.

::

    python_install() {
        distutils-r1_python_install

        # ensure data files for tests are getting installed too
        python_moduleinto collada/tests/
        python_domodule collada/tests/data
    }

``python_install_all`` installs documentation via ``einstalldocs``.
It is usually defined by ebuilds to install additional common files
such as bash completions or examples.

::

    python_install_all() {
        if use examples; then
            docinto examples
            dodoc -r Sample_Code/.
            docompress -x /usr/share/doc/${PF}/examples
        fi
        distutils-r1_python_install_all
    }


.. index:: esetup.py

Calling custom setup.py commands
================================
When working on packages using setuptools or modified distutils, you
sometimes need to manually invoke ``setup.py``.  The eclass provides
a ``esetup.py`` helper that wraps it with additional checks, error
handling and ensures that the override configuration file is created
beforehand (much like ``econf`` or ``emake``).

``esetup.py`` passes all its paremeters to ``./setup.py``.

::

    python_test() {
        esetup.py check
    }


Preventing test directory from being installed
==============================================
Many packages using the setuptools build system utilize the convenient
``find_packages()`` method to locate the Python sources.  In some cases,
this method also wrongly grabs top-level test directories or other files
that were not intended to be installed.

The eclass attempts to detect and report the most common mistakes:

.. code-block:: console

     *   Package installs 'tests' package which is forbidden and likely a bug in the build system.

The correct fix for this problem is to add an ``exclude`` parameter
to the ``find_packages()`` call in ``setup.py``, e.g.:

.. code-block:: python

    setup(
        packages=find_packages(exclude=["tests", "tests.*"]))

Note that if the top-level ``tests`` package has any subpackages, both
``tests`` and ``tests.*`` need to be listed.

.. warning::

   In order to test your fix properly, you need to remove the previous
   build directory.  Otherwise, the install command will install all
   files found there, including the files that are now excluded.

As an intermediate solution it is possible to strip the extra
directories in the install phase::

    python_install() {
        rm -r "${BUILD_DIR}"/lib/tests || die
        distutils-r1_python_install
    }


.. index:: distutils_enable_tests

Enabling tests
==============
Since Python performs only minimal build-time (or more precisely,
import-time) checking of correctness, it is important to run tests
of Python packages in order to catch any problems early.  This is
especially important for permitting others to verify support for new
Python implementations.

Many Python packages use one of the standard test runners, and work fine
with the default ways of calling them.  Note that upstreams sometimes
specify a test runner that's not strictly necessary — e.g. specify
``dev-python/pytest`` as a dependency while the tests do not use it
anywhere and work just fine with built-in modules.  The best way
to determine the test runner to use is to grep the test sources.


Using distutils_enable_tests
----------------------------
The simplest way of enabling tests is to call ``distutils_enable_tests``
in global scope, passing the test runner name as the first argument.
This function takes care of declaring test phase, setting appropriate
dependencies and ``test`` USE flag if necessary.  If called after
setting ``RDEPEND``, it also copies it to test dependencies.

.. code-block:: bash
   :emphasize-lines: 22

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 python3_{6,7,8} pypy3 )
    inherit distutils-r1

    DESCRIPTION="An easy whitelist-based HTML-sanitizing tool"
    HOMEPAGE="https://github.com/mozilla/bleach https://pypi.org/project/bleach/"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="Apache-2.0"
    SLOT="0"
    KEYWORDS="~alpha ~amd64 ~arm ~arm64 ~hppa ~ia64 ~mips ~ppc ~ppc64 ~s390 ~sparc ~x86"

    RDEPEND="
        dev-python/six[${PYTHON_USEDEP}]
        dev-python/webencodings[${PYTHON_USEDEP}]
    "

    distutils_enable_tests pytest

The valid values include:

- ``nose`` for ``dev-python/nose``
- ``pytest`` for ``dev-python/pytest``
- ``setup.py`` to call ``setup.py test``
- ``unittest`` to use built-in unittest discovery


Adding more test dependencies
-----------------------------
Additional test dependencies can be specified in ``test?`` conditional.
The flag normally does not need to be explicitly declared,
as ``distutils_enable_tests`` does that in the majority of cases.

.. code-block:: bash
   :emphasize-lines: 18,21

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6

    PYTHON_COMPAT=( python2_7 python3_{6,7,8} pypy3 )
    inherit distutils-r1

    DESCRIPTION="Universal encoding detector"
    HOMEPAGE="https://github.com/chardet/chardet https://pypi.org/project/chardet/"
    SRC_URI="https://github.com/chardet/chardet/archive/${PV}.tar.gz -> ${P}.tar.gz"

    LICENSE="LGPL-2.1"
    SLOT="0"
    KEYWORDS="~alpha amd64 arm arm64 hppa ia64 ~m68k ~mips ppc ppc64 s390 ~sh sparc x86 ~x64-cygwin ~amd64-linux ~x86-linux ~x64-macos ~x86-macos ~x64-solaris"

    DEPEND="
        test? ( dev-python/hypothesis[${PYTHON_USEDEP}] )
    "

    distutils_enable_tests pytest

Note that ``distutils_enable_tests`` modifies variables directly
in the ebuild environment.  This means that if you wish to change their
values, you need to append to them, i.e. the bottom part of that ebuild
can be rewritten as:

.. code-block:: bash
   :emphasize-lines: 3

    distutils_enable_tests pytest

    DEPEND+="
        test? ( dev-python/hypothesis[${PYTHON_USEDEP}] )
    "

Installing the package before running tests
-------------------------------------------
``distutils_enable_tests`` can also install the package to a temporary
directory before running tests.  To do that, pass ``--install``
as the first option.  Fore more information, see `installing the package
before testing`_.


Undesirable test dependencies
-----------------------------
There is a number of packages that are frequently listed as test
dependencies upstream but have little to no value for Gentoo users.
It is recommended to skip those test dependencies whenever possible.
If tests fail to run without them, it is often preferable to strip
the dependencies and/or configuration values enforcing them.

*Coverage testing* establishes how much of the package's code is covered
by the test suite.  While this is useful statistic upstream, it has
no value for Gentoo users who just want to install the package.  This
is often represented by dependencies on ``dev-python/coverage``,
``dev-python/pytest-cov``.  In the latter case, you usually need
to strip ``--cov`` parameter from ``pytest.ini``.

*PEP-8 testing* enforces standard coding style across Python programs.
Issues found by it are relevant to upstream but entirely irrelevant
to Gentoo users.  If the package uses ``dev-python/pep8``,
``dev-python/pycodestyle``, ``dev-python/flake8``, strip that
dependency.

``dev-python/pytest-runner`` is a thin wrapper to run pytest
from ``setup.py``.  Do not use it, just call pytest directly.

``dev-python/tox`` is a convenient wrapper to run tests for multiple
Python versions, in a virtualenv.  The eclass already provides the logic
for the former, and an environment close enough to the latter.  Do not
use tox in ebuilds.


Customizing the test phase
--------------------------
If additional pre-/post-test phase actions need to be performed,
they can be easily injected via overriding ``src_test()`` and making
it call ``distutils-r1_src_test``:

.. code-block:: bash
   :emphasize-lines: 30-34

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7,8} )
    inherit distutils-r1 virtualx

    DESCRIPTION="Extra features for standard library's cmd module"
    HOMEPAGE="https://github.com/python-cmd2/cmd2"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="MIT"
    SLOT="0"
    KEYWORDS="~amd64 ~arm ~arm64 ~ppc64 ~x86 ~amd64-linux ~x86-linux"

    RDEPEND="
        dev-python/attrs[${PYTHON_USEDEP}]
        >=dev-python/colorama-0.3.7[${PYTHON_USEDEP}]
        >=dev-python/pyperclip-1.6[${PYTHON_USEDEP}]
        dev-python/six[${PYTHON_USEDEP}]
        dev-python/wcwidth[${PYTHON_USEDEP}]
    "
    BDEPEND="
        dev-python/setuptools_scm[${PYTHON_USEDEP}]
    "

    distutils_enable_tests pytest

    src_test() {
        # tests rely on very specific text wrapping...
        local -x COLUMNS=80
        virtx distutils-r1_src_test
    }

If the actual test command needs to be customized, or a non-standard
test tool needs to be used, you can define a ``python_test()`` sub-phase
function.  This function is called for every enabled Python target
by the default ``src_test`` implementation.  This can either be combined
with ``distutils_enable_tests`` call, or used instead of it.  In fact,
the former function simply defines a ``python_test()`` function as part
of its logic.

.. code-block:: bash
   :emphasize-lines: 16,17,26-31,33-35

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python{2_7,3_6,3_7,3_8} pypy3 )
    inherit distutils-r1

    DESCRIPTION="Bash tab completion for argparse"
    HOMEPAGE="https://pypi.org/project/argcomplete/"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="Apache-2.0"
    SLOT="0"
    KEYWORDS="~amd64 ~arm ~arm64 ~hppa ~x86 ~amd64-linux ~x86-linux ~x64-macos"
    IUSE="test"
    RESTRICT="!test? ( test )"

    RDEPEND="
        $(python_gen_cond_dep '
            <dev-python/importlib_metadata-2[${PYTHON_USEDEP}]
        ' -2 python3_{5,6,7} pypy3)"
    # pip is called as an external tool
    BDEPEND="
        dev-python/setuptools[${PYTHON_USEDEP}]
        test? (
            app-shells/fish
            app-shells/tcsh
            dev-python/pexpect[${PYTHON_USEDEP}]
            dev-python/pip
        )"

    python_test() {
        "${EPYTHON}" test/test.py -v || die
    }

Note that ``python_test`` is called by ``distutils-r1_src_test``,
so you must make sure to call it if you override ``src_test``.


.. index:: epytest

Customizing the test phase for pytest
-------------------------------------
For the relatively frequent case of pytest-based packages needing
additional customization, a ``epytest`` helper is provided.  The helper
runs ``pytest`` with a standard set of options and automatic handling
of test failures.

For example, if upstream uses ``network`` marker to disable
network-based tests, you can override the test phase in the following
way::

    distutils_enable_tests pytest

    python_test() {
        epytest -m 'not network'
    }


.. index:: distutils_install_for_testing

Installing the package before testing
-------------------------------------
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

This function currently supports three install layouts:

- the standard *root directory* layout that is enabled
  via ``--via-root``,

- a virtualenv-alike *venv* layout that is enabled via ``--via-venv``,

- the legacy *home directory* layout that is enabled via ``--via-home``
  parameter.


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

If neither of the two works, you may try forcing the legacy layout
via ``--via-home``.  However, if you need to do that, please report
a bug for the eclass, so that we can look for a better solution looking
forward.  This layout is planned on being removed in EAPI 8.

The home directory layout historically used to be necessary to fix
problems with some packages.  However, the underlying issues probably
went away along with old versions of Python, and the `removal of site.py
hack`_ has broken it for most of the consumers.

.. code-block:: bash

    python_test() {
        distutils_install_for_testing
        epytest --no-network
    }


.. index:: distutils_enable_sphinx

Building documentation via Sphinx
=================================
``dev-python/sphinx`` is commonly used to document Python packages.
It comes with a number of plugins and themes that make it convenient
to write and combine large text documents (such as this Guide!),
as well as automatically document Python code.

Depending on the exact package, building documentation may range
from being trivial to very hard.  Packages that do not use autodoc
(documenting of Python code) do not need to USE-depend on Sphinx at all.
Packages that do that need to use a supported Python implementation
for Sphinx, and packages that use plugins need to guarantee the same
implementation across all plugins.  To cover all those use cases easily,
the ``distutils_enable_sphinx`` function is provided.


Basic documentation with autodoc
--------------------------------
The most common case is a package that uses Sphinx along with autodoc.
It can be recognized by ``conf.py`` listing ``sphinx.ext.autodoc``
in the extension list.  In order to support building documentation,
call ``distutils_enable_sphinx`` and pass the path to the directory
containing Sphinx documentation:

.. code-block:: bash
   :emphasize-lines: 24

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7,8} )
    DISTUTILS_USE_SETUPTOOLS=rdepend

    inherit distutils-r1

    DESCRIPTION="Colored stream handler for the logging module"
    HOMEPAGE="
        https://pypi.org/project/coloredlogs/
        https://github.com/xolox/python-coloredlogs
        https://coloredlogs.readthedocs.io/en/latest/"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="MIT"
    SLOT="0"
    KEYWORDS="~amd64 ~x86 ~amd64-linux ~x86-linux"

    RDEPEND="dev-python/humanfriendly[${PYTHON_USEDEP}]"

    distutils_enable_sphinx docs

This call takes care of it all: it adds ``doc`` USE flag to control
building documentation, appropriate dependencies via the expert any-r1
API making it sufficient for Sphinx to be installed with only one
of the supported implementations, and appropriate ``python_compile_all``
implementation to build and install HTML documentation.


Additional Sphinx extensions
----------------------------
It is not uncommon for packages to require additional third-party
extensions to Sphinx.  Those include themes.  In order to specify
dependencies on the additional packages, pass them as extra arguments
to ``distutils_enable_sphinx``.

.. code-block:: bash
   :emphasize-lines: 17-20

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( pypy3 python3_{6,7,8} )
    inherit distutils-r1

    DESCRIPTION="Correctly inflect words and numbers"
    HOMEPAGE="https://github.com/jazzband/inflect"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="MIT"
    SLOT="0"
    KEYWORDS="~amd64 ~arm64 ~ia64 ~ppc ~ppc64 ~x86"

    distutils_enable_sphinx docs \
        '>=dev-python/jaraco-packaging-3.2' \
        '>=dev-python/rst-linker-1.9' \
        dev-python/alabaster

In this case, the function uses the any-r1 API to request one
of the supported implementations to be enabled on *all* of those
packages.  However, it does not have to be the one in ``PYTHON_TARGETS``
for this package.


Sphinx without autodoc or extensions
------------------------------------
Finally, there are packages that use Sphinx purely to build
documentation from text files, without inspecting Python code.
For those packages, the any-r1 API can be omitted entirely and plain
dependency on ``dev-python/sphinx`` is sufficient.  In this case,
the ``--no-autodoc`` option can be specified instead of additional
packages.

.. code-block:: bash
   :emphasize-lines: 17

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 python3_{6,7,8} )
    inherit distutils-r1

    DESCRIPTION="Python Serial Port extension"
    HOMEPAGE="https://github.com/pyserial/pyserial https://pypi.org/project/pyserial/"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="PSF-2"
    SLOT="0"
    KEYWORDS="~alpha amd64 ~arm arm64 ~hppa ~ia64 ~m68k ~mips ~ppc ~ppc64 ~s390 ~sh ~sparc ~x86"

    distutils_enable_sphinx documentation --no-autodoc

Note that this is valid only if no third-party extensions are used.
If additional packages need to be installed, the previous variant
must be used instead.

The eclass tries to automatically determine whether ``--no-autodoc``
should be used, and issue a warning if it's missing or incorrect.


.. _distutils-r1.eclass(5):
   https://devmanual.gentoo.org/eclass-reference/distutils-r1.eclass/index.html
.. _removal of site.py hack:
   https://github.com/pypa/setuptools/commit/91213fb2e7eecde9f5d7582de485398f546e7aa8
