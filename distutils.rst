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
The most basic example presented above assumes that the package is using
``dev-python/setuptools`` build system and not installing entry points.
The eclass automatically tries to detect whenever the default might be
incorrect, and reports it::

     * DISTUTILS_USE_SETUPTOOLS value is probably incorrect
     *   value:    DISTUTILS_USE_SETUPTOOLS=bdepend (default?)
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

This example package installs *entry points* (grep for ``entry_points``
in ``setup.py`` or ``setup.cfg``), that is wrapper scripts that use
setuptools to import the installed package files and execute subroutines
within them.  This means that ``dev-python/setuptools`` must remain
installed at runtime and the variable is set to ``rdepend``.

Some packages do not use setuptools at all, and instead use plain
distutils.  In this case, the correct value is simply ``no``.

There are a few cases where the automatic check does not yield correct
values, e.g. when ``dev-python/setuptools`` is used at runtime in other
way than through entry points.  In that case, a value of ``manual``
can be used to disable the logic completely and specify the dependencies
manually.

When a ``pyproject.toml``-based build system (flit, poetry) is being
used, a value of ``pyproject.toml`` can be used to enable installing it
via setuptools.  For this purpose, a build-time dependency
on ``dev-python/pyproject2setuppy`` is added.


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
specifically to some plugin systems or packages using
the ``pkg_resources`` framework.

The ``distutils_install_for_testing`` function runs ``setup.py install``
into a temporary directory, and adds the appropriate paths to ``PATH``
and ``PYTHONPATH``.  It uses the *home directory* install layout that
should satisfy most of the remaining test suites.

.. code-block:: bash

    python_test() {
        distutils_install_for_testing
        pytest -vv --no-network || die "Testsuite failed under ${EPYTHON}"
    }

Note that ``distutils_install_for_testing`` is quite a heavy hammer.
It is useful for solving hard cases and initially determining the cause
of failing tests.  However, many packages will be entirely satisfied
with simpler solutions, such as changing the working directory
to ``${BUILD_DIR}/lib``.


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
