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


The PEP 517 and legacy modes
============================
.. Warning::

   The PEP 517 mode is still experimental and it is not guaranteed
   to handle all packages correctly.  When using it, please verify
   that all necessary files are installed correctly.  The hooks provided
   by ``app-portage/iwdevtools`` can be helpful in checking for
   regressions when migrating existing packages.

The ``distutils-r1`` eclass has currently two modes of operation:
the PEP 517 mode and the legacy mode.  The former mode should be
preferred for new ebuilds; the latter is provided for backwards
compatibility and packages that are not PEP 517-ready.

The PEP 517 mode uses backends as defined by `PEP 517`_ to build
packages.  It supports a greater number of Python build systems
at the cost of flexibility and performance.  In the eclass
implementation, the PEP 517 backend is used to build a wheel (i.e. a zip
archive) with the package and then an installer tool is used to install
the wheel into a staging directory.  The complete process is done
in compile phase, and the install phase merely moves the files into
the image directory.

The legacy mode invokes the ``setup.py`` script directly.  The build
command is invoked to populate the build directory in the compile phase,
then the install command is used in the install phase.  Normally, this
mode works only for packages using backwards-compatible distutils
derivatives.  Additionally, it supports flit and poetry through
pyproject2setuppy hack.  This mode relies on deprecated features.

The PEP 517 mode is enabled via declaring the ``DISTUTILS_USE_PEP517``
variable.  Otherwise, the legacy mode is used.


Basic use (PEP 517 mode)
========================
By default, ``distutils-r1`` sets appropriate metadata variables
and exports a full set of phase functions necessary to install packages
using Python build systems.

The ``DISTUTILS_USE_PEP517`` variable is used to enable the modern
PEP 517 mode and declare the build system used.  The eclass
automatically generates a build-time dependency on the packages needed
for the build system.

The simplest case of ebuild is:

.. code-block:: bash
   :emphasize-lines: 6-8

    # Copyright 1999-2022 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=8

    DISTUTILS_USE_PEP517=setuptools
    PYTHON_COMPAT=( python3_{8..10} pypy3 )
    inherit distutils-r1

    DESCRIPTION="Makes working with XML feel like you are working with JSON"
    HOMEPAGE="https://github.com/martinblech/xmltodict/ https://pypi.org/project/xmltodict/"
    SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"

    LICENSE="MIT"
    SLOT="0"
    KEYWORDS="~amd64 ~arm ~arm64 ~x86"


Source archives
===============
The vast majority of Python packages can be found in the `Python Package
Index (PyPI)`_.  Often this includes both source (sdist) and binary
(wheel) packages.  In addition to that, many packages have public VCS
repositories with an automatic archive generation mechanism
(e.g. GitHub).

The current recommendation is to prefer these *generated archives*
(snapshots) over official sdist archives.  This is because sdist
archives often miss files that are not strictly required for binary
distribution.  This usually includes tests, test data files,
documentation but historically there were also instances of sdist
releases that were entirely nonfunctional.

When using generated archives, it is recommended to append a unique
suffix (in case of GitHub, using a ``.gh.tar.gz`` suffix is requested)
to the distfile name, in order to make the archive clearly
distinguishable from the upstream provided tarball and to use a filename
that matches the top directory inside the archive, e.g.:

.. code-block:: bash

    SRC_URI="
        https://github.com/Textualize/rich/archive/v${PV}.tar.gz
            -> ${P}.gh.tar.gz
    "

Note that unlike sdist archives, snapshots are often missing generated
files.  This has some implications, notably:

1. If the package uses setuptools_scm, the version string needs
   to be provided explicitly, cf. `setuptools_scm and snapshots`_.

2. If the package uses Cython, the C files need to be generated
   and an explicit ``BDEPEND`` on ``dev-python/cython`` needs to
   be added.  However, regenerating them is recommended anyway,
   cf. `packages using Cython`_.

Nevertheless, in some cases sdist archives (or even a combination
of both archive kinds) will be preferable because of pregenerated files
that may require Internet access or have problematic dependencies
(e.g. NodeJS).

.. _Python Package Index (PyPI): https://pypi.org/


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
   :emphasize-lines: 9

    # Copyright 1999-2022 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{8..10} )
    PYTHON_REQ_USE="readline"
    DISTUTILS_USE_PEP517=setuptools
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


.. index:: DISTUTILS_USE_PEP517

PEP 517 build systems
=====================
The majority of examples in this guide assume using setuptools build
system.  However, PEP 517 mode provides support for other build systems.

In order to determine the correct build system used, read
the ``pyproject.toml`` file.  An example file could start with:

.. code-block:: toml

    [build-system]
    requires = ["flit_core >=3.6.0,<4"]
    build-backend = "flit_core.buildapi"

The ``requires`` key indicates the packages required in order to run
the build system, while ``build-backend`` indicates the module
(and optionally the class) providing the build system backend.
The eclass maintains a mapping of backend paths to the respective
``DISTUTILS_USE_PEP517`` and automatically suggests the correct value.

The following table summarizes supported backends.

================== ============================ ================================
USE_PEP517 value   Provider package             build-backend
================== ============================ ================================
flit               dev-python/flit_core         flit_core.buildapi
flit_scm           dev-python/flit_scm          flit_scm:buildapi
hatchling          dev-python/hatchling         hatchling.build
jupyter            dev-python/jupyter_packaging jupyter_packaging.build_api
maturin            dev-util/maturin             maturin
pbr                dev-python/pbr               pbr
pdm                dev-python/pdm-pep517        pdm.pep517.api
poetry             dev-python/poetry-core       poetry.core.masonry.api
setuptools         dev-python/setuptools        setuptools.build_meta
                                                setuptools.__legacy__.build_meta

sip                dev-python/sip               sipbuild.api
standalone         (none)                       (various)
================== ============================ ================================

The special value ``standalone`` is reserved for bootstrapping build
systems.  It indicates that the package itself provides its own build
backend.

Legacy packages that provide ``setup.py`` but no ``pyproject.toml``
(or do not define a backend inside it) should be installed via
the ``setuptools`` backend (this applies to pure distutils packages
as well).  The eclass automatically uses the legacy setuptools backend
for them.


Deprecated PEP 517 backends
===========================

flit.buildapi
-------------
Some packages are still found using the historical flit build backend.
Their ``pyproject.toml`` files contain a section similar
to the following:

.. code-block:: toml

    [build-system]
    requires = ["flit"]
    build-backend = "flit.buildapi"

This backend requires installing the complete flit package manager.
Instead, the package should be fixed upstream to use flit_core
per `flit build system section documentation`_ instead:

.. code-block:: toml

    [build-system]
    requires = ["flit_core"]
    build-backend = "flit_core.buildapi"

flit_core produces identical artifacts to flit.  At the same time, it
reduces the build-time dependency footprint and therefore makes isolated
PEP 517 builds faster.


poetry.masonry.api
------------------
A similar problem applies to the packages using poetry.  The respective
``pyproject.toml`` files contain:

.. code-block:: toml

    [build-system]
    requires = ["poetry>=0.12"]
    build-backend = "poetry.masonry.api"

Instead, the lightweight poetry-core module should be used per `poetry
PEP-517 documentation`_:

.. code-block:: toml

    [build-system]
    requires = ["poetry_core>=1.0.0"]
    build-backend = "poetry.core.masonry.api"

poetry-core produces identical artifacts to poetry.  It has smaller
dependency footprint and makes isolated builds much faster.


setuptools.build_meta:__legacy__
--------------------------------
Some packages using setuptools specify the following:

.. code-block:: toml

    [build-system]
    requires = ["setuptools>=40.8.0", "wheel"]
    build-backend = "setuptools.build_meta:__legacy__"

This is incorrect, as the legacy backend is intended to be used only
as an implicit fallback.  All packages should be using the regular
backend instead:

.. code-block:: toml

    [build-system]
    requires = ["setuptools>=40.8.0"]
    build-backend = "setuptools.build_meta"

Please also note that the ``wheel`` package should *not* be listed
as a dependency, as it is an implementation detail and it was always
implicitly returned by the backend.  Unfortunately, due to prolonged
documentation error, a very large number of packages still specifies it,
and other packages tend to copy that mistake.


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


.. index:: Cython

Packages using Cython
=====================
Cython_ is a static compiler that permits writing Python extensions
in a hybrid of C and Python.  Cython files are compiled into C code
that is compatible with multiple Python interpreters.  This makes it
possible for packages to include pregenerated C files and build
the respective extensions without exposing the Cython dependency.

In Gentoo, it is always recommended to depend on ``dev-python/cython``
and regenerate the C files.  This guarantees that bug fixes found
in newer versions of Cython are taken advantage of.  Using shipped files
could e.g. cause compatibility issues with newer versions of Python.

Depending on the package in question, forcing regeneration could be
as simple as removing the pregenerated files:

.. code-block:: bash

    BDEPEND="
        dev-python/cython[${PYTHON_USEDEP}]
    "

    src_configure() {
        rm src/frobnicate.c || die
    }

However, in some cases packages utilize the generated C files directly
in ``setup.py``.  In these cases, sometimes a Makefile is provided
to run Cythonize.  It is also possible to call Cython directly:

.. code-block:: bash

    BDEPEND="
        dev-python/cython[${PYTHON_USEDEP}]
    "

    src_configure() {
        cython -3 jq.pyx -o jq.c || die
    }

Note that Cython needs to be called only once, as the resulting code
is compatible with all Python versions.

.. _Cython: https://cython.org/


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

In the following example, it is used to remove a CLI script whose
dependencies only support Python 3.8 and 3.9 at the moment.  Naturally,
since this modification needs to be done on a subset of all Python
interpreters, the eclass needs to keep a separate copy of the sources
for every one of them.  This is why ``python_prepare`` automatically
enables in-source builds.

::

    python_prepare() {
        if ! use cli || ! has "${EPYTHON}" python3.{7..9}; then
            sed -i -e '/console_scripts/d' setup.py || die
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
        DISTUTILS_ARGS=(
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


.. index:: DISTUTILS_ARGS

Passing arguments to setup.py
=============================
There are two main methods of accepting additional command-line options
in ``setup.py`` scripts: using global options and via command options.

Global options are usually implemented through manipulating ``sys.path``
directly.  The recommended way to use them is to specify them
via ``DISTUTILS_ARGS`` array::

    src_configure() {
        DISTUTILS_ARGS=( --external )
    }

The options specified via ``DISTUTILS_ARGS`` are passed to all
``esetup.py`` invocations, as well as to the setuptools PEP517 backend
(using the ``--global-option`` setting).  For future compatibility,
it is recommended to avoid adding command names to ``DISTUTILS_ARGS``.

The recommended way to pass command options is to use the ``setup.cfg``
file.  For example, Pillow provides for configuring available backends
via additional ``build_ext`` command flags::

    setup.py build_ext --enable-tiff --disable-webp ...

The respective options can be setup via the configuration file, where
sections represent the commands and individual keys — options.  Note
that dashes need to be replaced by underscores, and flag-style options
take boolean arguments.  In this case, the ebuild can use::

    src_configure() {
        cat >> setup.cfg <<-EOF
            [build_ext]
            disable_tiff = $(usex !tiff True False)
            enable_tiff = $(usex tiff True False)
            disable_webp = $(usex !webp True False)
            enable_webp = $(usex webp True False)
            #...
        EOF
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
- ``setup.py`` to call ``setup.py test`` (*deprecated*)
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
In PEP 517 mode, the eclass automatically exposes a venv-style install
tree to the test phase.  No explicit action in necessary.

In the legacy mode, ``distutils_enable_tests`` has an optional
``--install`` option that can be used to force performing an install
to a temporary directory.  More information can be found in the legacy
section.


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
    inherit distutils-r1

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
        distutils-r1_src_test
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


.. index:: virtx

Running tests with virtualx
---------------------------
Test suites requiring a display to work correctly can often be appeased
usng Xvfb.  If the package in question does not start Xvfb directly,
``virtualx.eclass`` can be used to do that.  Whenever possible, it is
preferable to run a single server in ``src_test()`` for all passes
of the test suite, e.g.::

    distutils_enable_tests pytest

    src_test() {
        virtx distutils-r1_src_test
    }

Note that ``virtx`` implicitly enables nonfatal mode.  This means that
e.g. ``epytest`` will no longer terminate the ebuild on failure, and you
need to use ``die`` explicitly for it::

    src_test() {
        virtx distutils-r1_src_test
    }

    python_test() {
        epytest -m "not network" || die "Tests failed with ${EPYTHON}"
    }

.. Warning::

   Explicit ``|| die`` is only necessary when overriding ``python_test``
   and running ``epytest`` inside a ``nonfatal``.  The ``virtx`` command
   runs its arguments via a ``nonfatal``.  The default ``python_test``
   implementation created by ``distutils_enable_tests`` accounts for
   this.  In other contexts, ``epytest`` will die on its own.


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


.. index:: DISTUTILS_DEPS
.. index:: DISTUTILS_OPTIONAL

Packages with optional Python build system usage
================================================
The eclass has been written with the assumption that the vast majority
of its consumers will be using the Python build systems unconditionally.
For this reason, it sets the ebuild metadata variables (dependencies,
``REQUIRED_USE``) and exports phase functions by default.  However, it
also provides support for *optional mode* that can be used when Python
is used conditionally to USE flags.

If ``DISTUTILS_OPTIONAL`` is set to a non-empty value, then the eclass
does not alter ebuild metadata or export phase functions by default.
The ebuild needs to declare appropriate dependencies
and ``REQUIRED_USE`` explicitly, and call the appropriate phase
functions.

The ``PYTHON_DEPS`` and ``PYTHON_REQUIRED_USE`` variables provided
by the underlying Python eclasses should be used, as if using these
eclasses directly.  Furthermore, in PEP 517 mode an additional
``DISTUTILS_DEPS`` variable is exported that contains build-time
dependnecies specific to wheel build and install, and should be added
to ``BDEPEND``.

At the very least, the phases having default `sub-phase functions`_ need
to be called, that is:

- ``distutils-r1_src_prepare``
- ``distutils-r1_src_compile``
- ``distutils-r1_src_install``

Additional phases need to be called if the ebuild declares sub-phase
functions for them.

Note that in optional mode, the default implementation
of ``distutils-r1_python_prepare_all`` does not apply patches (to avoid
collisions with other eclasses).

.. Warning::

   The ``distutils_enable_sphinx`` and ``distutils_enable_tests`` alter
   the ebuild metadata variables and declare sub-phase functions
   independently of the value of ``DISTUTILS_OPTIONAL``.  However,
   in order for the respective sub-phases to be executed the ebuild
   needs to call appropriate eclass phase functions (i.e. additionally
   call ``distutils-r1_src_test`` for the latter).

   If unconditional test dependencies are undesirable, these functions
   cannot be used, and appropriate dependencies and sub-phases need
   to be declared explicitly.

   In the legacy mode, the ``DISTUTILS_USE_SETUPTOOLS`` variable is
   not used if the optional mode is enabled.  Instead, the dependency
   on ``dev-python/setuptools`` needs to be declared explicitly.

An example ebuild for a package utilizing autotools as a primary build
system alongside a flit-based ``pyproject.toml`` in the top directory
follows:

.. code-block:: bash
   :emphasize-lines: 6-10,13-15,21-24,26-33,37,42,45-47,51,56

    # Copyright 1999-2022 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=8

    DISTUTILS_USE_PEP517=flit
    DISTUTILS_OPTIONAL=1
    PYTHON_COMPAT=( python3_{8..10} pypy3 )

    inherit distutils-r1

    # ...
    IUSE="python test"
    REQUIRED_USE="
        python? ( ${PYTHON_REQUIRED_USE} )"

    DEPEND="
        dev-libs/libfoo:="
    RDEPEND="
        ${DEPEND}
        python? (
            ${PYTHON_DEPS}
            dev-python/frobnicate[${PYTHON_USEDEP}]
        )"
    BDEPEND="
        python? (
            ${PYTHON_DEPS}
            ${DISTUTILS_DEPS}
            test? (
                dev-python/frobnicate[${PYTHON_USEDEP}]
                dev-python/pytest[${PYTHON_USEDEP}]
            )
        )"

    src_prepare() {
        default
        use python && distutils-r1_src_prepare
    }

    src_compile() {
        default
        use python && distutils-r1_src_configure
    }

    python_test() {
        epytest
    }

    src_test() {
        default
        use python && distutils-r1_src_test
    }

    src_install() {
        default
        use python && distutils-r1_src_install
    }


.. index:: Rust

Packages with Rust extensions (using Cargo)
===========================================
Some Python build systems include support for writing extensions
in the Rust programming language.  Two examples of these are setuptools
using ``dev-python/setuptools_rust`` plugin and Maturin.  Normally,
these build systems utilize the Cargo ecosystem to automatically
download the Rust dependencies over the Internet.  In Gentoo,
``cargo.eclass`` is used to provide these dependencies to ebuilds.

When creating a new ebuild for a package using Rust extensions
or bumping one, you need to locate the ``Cargo.lock`` files within
the package's sources.  For each of these files, run ``cargo-ebuild``
in the containing directory in order to generate a template ebuild.
Then combine ``CRATES`` and ``LICENSE`` values from all the generated
ebuilds.

The actual ebuild inherits both ``cargo`` and ``distutils-r1`` eclasses.
Prior to inherit, ``CARGO_OPTIONAL`` should be used to avoid exporting
phase functions, and ``CRATES`` should be declared.  ``SRC_URI`` needs
to contain URLs generated using ``cargo_crate_uris``, and ``LICENSE``
the crate licenses in addition to the Python package's license.
``QA_FLAGS_IGNORED`` needs to match all Rust extensions in order
to prevent false positives on ignored ``CFLAGS`` and ``LDFLAGS``
warnings.  Finally, the ebuild needs to call ``cargo_src_unpack``.

An example ebuild follows:

.. code-block:: bash
   :emphasize-lines: 6,10-15,17,23,28,31,35,38

    # Copyright 2022 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=8

    CARGO_OPTIONAL=1
    DISTUTILS_USE_PEP517=setuptools
    PYTHON_COMPAT=( python3_{8..10} pypy3 )

    CRATES="
        Inflector-0.11.4
        aliasable-0.1.3
        asn1-0.8.7
        asn1_derive-0.8.7
    "

    inherit cargo distutils-r1

    # ...

    SRC_URI="
        mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz
        $(cargo_crate_uris ${CRATES})
    "

    LICENSE="|| ( BSD-2 Apache-2 )"
    # Crate licenses
    LICENSE+=" Apache-2.0 BSD BSD-2 MIT"

    BDEPEND="
        dev-python/setuptools-rust[${PYTHON_USEDEP}]
    "

    # Rust does not respect CFLAGS/LDFLAGS
    QA_FLAGS_IGNORED=".*/_rust.*"

    src_unpack() {
        cargo_src_unpack
    }


.. _distutils-r1.eclass(5):
   https://devmanual.gentoo.org/eclass-reference/distutils-r1.eclass/index.html
.. _PEP 517:
   https://www.python.org/dev/peps/pep-0517/
.. _flit build system section documentation:
   https://flit.readthedocs.io/en/latest/pyproject_toml.html#build-system-section
.. _poetry PEP-517 documentation:
   https://python-poetry.org/docs/pyproject/#poetry-and-pep-517
