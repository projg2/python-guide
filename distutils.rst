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
The ``distutils-r1`` eclass has currently two modes of operation:
the PEP 517 mode and the legacy mode.  The former mode should be
preferred for new ebuilds; the latter is provided for backwards
compatibility and packages that are incompatible with the other mode.

The PEP 517 mode uses backends as defined by `PEP 517`_ to build
packages.  It supports a greater number of Python build systems
at the cost of flexibility and performance.  In the eclass
implementation, the PEP 517 backend is used to build a wheel (i.e. a zip
archive) with the package and then an installer tool is used to install
the wheel into a staging directory.  The complete process is done
in compile phase, and the install phase merely moves the files into
the image directory.

The PEP 517 mode also features a 'no build system' mode for packages
that do not or cannot use a PEP 517-compliant build backend.

The legacy mode invokes the ``setup.py`` script directly.  The build
command is invoked to populate the build directory in the compile phase,
then the install command is used in the install phase.  Normally, this
mode works only for packages using backwards-compatible distutils
derivatives.  Additionally, it supports flit and poetry through
pyproject2setuppy hack.  This mode relies on deprecated features.

The PEP 517 mode is enabled via declaring the ``DISTUTILS_USE_PEP517``
variable.  The legal values can be found in the `PEP 517 build
systems`_ section.  If unset, the legacy mode is used.


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


.. _source archives:

Source archives
===============
The vast majority of Python packages can be found in the `Python Package
Index (PyPI)`_.  Often this includes both source (sdist) and binary
(wheel) packages.  In addition to that, many packages have public VCS
repositories with an automatic archive generation mechanism
(e.g. GitHub).

The current recommendation is to prefer source distributions from PyPI
*if and only if* they include all files needed for the package,
especially tests.  If the PyPI distribution is missing some files,
VCS generated archives should be used instead.  In some extreme cases,
it may be necessary to use both and combine the files contained in them
(e.g. to combine files pregenerated using NodeJS from sdist with tests
from GitHub archive).

When fetching archives from PyPI, ``pypi.eclass`` should be used.
It is documented in its own chapter: :doc:`pypi`.

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

1. If the package uses setuptools_scm or a similar package, the version
   string needs to be provided explicitly,
   cf. `setuptools_scm (flit_scm, hatch-vcs) and snapshots`_.

2. If the package uses Cython, the C files need to be generated
   and an explicit ``BDEPEND`` on ``dev-python/cython`` needs to
   be added.  However, regenerating them is recommended anyway,
   cf. `packages using Cython`_.

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
meson-python       dev-python/meson-python      mesonpy
no                 (none)                       (none, see below)
pbr                dev-python/pbr               pbr
pdm                dev-python/pdm-pep517        pdm.pep517.api
poetry             dev-python/poetry-core       poetry.core.masonry.api
setuptools         dev-python/setuptools        setuptools.build_meta
                                                setuptools.__legacy__.build_meta

sip                dev-python/sip               sipbuild.api
standalone         (none)                       (various, see below)
================== ============================ ================================

The eclass recognizes two special values: ``no`` and ``standalone``.
``no`` is used to enable 'no build system' mode as described
in `installing packages without a PEP 517 build backend`_.
``standalone`` indicates that the package itself provides its own build
backend.

Legacy packages that provide ``setup.py`` but no ``pyproject.toml``
(or do not define a backend inside it) should be installed via
the ``setuptools`` backend (this applies to pure distutils packages
as well).  The eclass automatically uses the legacy setuptools backend
for them.


.. index:: SETUPTOOLS_SCM_PRETEND_VERSION
.. index:: flit_scm
.. index:: hatch-vcs
.. index:: setuptools_scm

setuptools_scm (flit_scm, hatch-vcs) and snapshots
==================================================
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

The flit_scm_ and hatch-vcs_ packages are both built on top
of setuptools_scm.  The same approach applies to both of them.

.. Warning::

   While ``SETUPTOOLS_SCM_PRETEND_VERSION`` is sufficient to make
   the package build, setuptools may install incomplete set of package
   data files.  Please take special care to verify that all files are
   installed.

.. _setuptools_scm: https://pypi.org/project/setuptools-scm/
.. _flit_scm: https://pypi.org/project/flit_scm/
.. _hatch-vcs: https://pypi.org/project/hatch-vcs/


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
3. ``python_configure`` (for ``src_configure``, for each impl.)
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
``esetup.py`` invocations, as well as to the setuptools PEP 517 backend
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


Enabling tests
==============
The support for test suites is now covered in the :doc:`test` chapter.


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
        use python && distutils-r1_src_compile
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
the package's sources.  Run ``pycargoebuild`` passing the list of
the containing directories to generate a template ebuild, e.g.::

    pycargoebuild /tmp/portage/dev-python/setuptools-rust-1.5.2/work/setuptools-rust-1.5.2/examples/*/

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

    LICENSE="|| ( BSD-2 Apache-2.0 )"
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


Installing packages without a PEP 517 build backend
===================================================
The eclass features a special 'no build system' that is dedicated
to packages that could benefit from distutils-r1 features yet either
do not use a PEP 517-compliant build system, or cannot use one.  This
generally means that either:

- it uses a non-PEP 517 build system (autotools, CMake, plain Meson)

- it does not feature a build system at all

- its build system cannot be used as that would cause cyclic
  dependencies during build backend bootstrap

This mode is not supposed to be used for legacy use of distutils or
setuptools — these are handled via the setuptools backend.

The use cases for this mode partially overlap with the use of other
Python eclasses, particularly python-single-r1.  Using distutils-r1
is recommended if one of the eclass features benefits the particular
ebuild, e.g. if Python modules are installed or one of the supported
test runners are used.  For pure bundles of Python scripts,
python-single-r1 is preferable.

The 'no build system' mode is enabled via setting the following value:

.. code-block:: bash

    DISTUTILS_USE_PEP517=no

When this mode is used, the following applies:

- no dependencies on a build backend or PEP 517 machinery are declared
  (``DISTUTILS_DEPS`` are empty)

- the default implementation, ``distutils-r1_python_compile`` is a no-op

However, the following eclass features are still available:

- Python interpreter dependencies, ``REQUIRED_USE`` and distutils-r1
  phase functions are used (unless disabled via ``DISTUTILS_OPTIONAL``)

- the temporary venv is created in ``${BUILD_DIR}/install`` for test
  phase to use (but the ebuild needs to install files there explicitly)

- the contents of ``${BUILD_DIR}/install`` are merged into ``${D}``
  by ``distutils-r1_python_install`` (if present; temporary venv files
  are removed)

- ``distutils_enable_sphinx`` and ``distutils_enable_tests``
  are functional


Installing packages manually into BUILD_DIR
-------------------------------------------
The simplest approach towards installing packages manually is to use
``python_domodule`` in ``python_compile`` sub-phase.  This causes
the modules to be installed into ``${BUILD_DIR}/install`` tree,
effectively enabling them to be picked up for the test phase
and merged in ``distutils-r1_python_install``.

An example ebuild using a combination of GitHub archive (for tests)
and PyPI wheel (for generated .dist-info) follows:

.. code-block:: bash
   :emphasize-lines: 3,19,22

    EAPI=7

    DISTUTILS_USE_PEP517=no
    PYTHON_COMPAT=( python3_{8..11} pypy3 )

    inherit distutils-r1

    SRC_URI="
        https://github.com/hukkin/tomli/archive/${PV}.tar.gz
            -> ${P}.gh.tar.gz
        https://files.pythonhosted.org/packages/py3/${PN::1}/${PN}/${P}-py3-none-any.whl
            -> ${P}-py3-none-any.whl.zip
    "

    BDEPEND="
        app-arch/unzip
    "

    distutils_enable_tests unittest

    python_compile() {
        python_domodule src/tomli "${WORKDIR}"/*.dist-info
    }

Note that the wheel suffix is deliberately changed in order to enable
automatic unpacking by the default ``src_unpack``.


Installing packages manually into D
-----------------------------------
The alternative approach is to install files in ``python_install``
phase.  This provides a greater number of helpers.  However,
the installed modules will not be provided in the venv for the test
phase.

An example ebuild follows:

.. code-block:: bash
   :emphasize-lines: 3,8,11-17

    EAPI=7

    DISTUTILS_USE_PEP517=no
    PYTHON_COMPAT=( pypy3 python3_{8..11} )

    inherit distutils-r1

    distutils_enable_tests pytest

    python_install() {
        python_domodule gpep517
        python_newscript - gpep517 <<-EOF
            #!${EPREFIX}/usr/bin/python
            import sys
            from gpep517.__main__ import main
            sys.exit(main())
        EOF
    }

It is also valid to combine both approaches, e.g. install Python modules
in ``python_compile``, and scripts in ``python_install``.  In this case,
``distutils-r1_python_install`` needs to be called explicitly.


Integrating with a non-PEP 517 build system
-------------------------------------------
The 'no build system' mode can also be used to use distutils-r1
sub-phases to integrate with a build system conveniently.  The following
ebuild fragment demonstrates using it with Meson:

.. code-block:: bash

    EAPI=8

    DISTUTILS_USE_PEP517=no
    PYTHON_COMPAT=( python3_{8..10} )

    inherit meson distutils-r1

    python_configure() {
        local emesonargs=(
            -Dlint=false
        )

        meson_src_configure
    }

    python_compile() {
        meson_src_compile
    }

    python_test() {
        meson_src_test
    }

    python_install() {
        meson_src_install
    }


.. _distutils-r1.eclass(5):
   https://devmanual.gentoo.org/eclass-reference/distutils-r1.eclass/index.html
.. _PEP 517:
   https://www.python.org/dev/peps/pep-0517/
