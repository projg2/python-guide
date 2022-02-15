======================
Expert python-r1 usage
======================

.. highlight:: bash

The APIs described in this chapter are powerful but even harder to use
than those described in ``python-r1`` chapter.  You should not consider
using them unless you have a proper ninja training.


.. index:: python_gen_useflags

Partially restricting Python implementation
===========================================
There are packages that have been ported to Python 3 only partially.
They may still have some optional dependencies that support Python 2
only, they may have some components that do not support Python 3 yet.
The opposite is also possible — some of the features being available
only for Python 3.

There are two approaches to this problem.  You can either skip features
(ignore USE flags) if the necessary implementation is not enabled,
or you can use ``REQUIRED_USE`` to enforce at least one interpreter
having the requested feature.

Skipping specific tasks can be done via investigating ``${EPYTHON}``.
If USE flags are involved, you will probably also need to use
``python_gen_cond_dep`` with additional parameters restricting
dependencies to available targets.

.. code-block:: bash
   :emphasize-lines: 37-43,49

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 python3_{6,7,8} pypy{,3} )
    PYTHON_REQ_USE="threads(+)"
    inherit distutils-r1

    DESCRIPTION="HTTP library for human beings"
    HOMEPAGE="http://python-requests.org/"
    SRC_URI="mirror://pypi/${P:0:1}/${PN}/${P}.tar.gz"

    LICENSE="Apache-2.0"
    SLOT="0"
    KEYWORDS="~amd64 ~arm ~arm64 ~sparc ~x86 ~amd64-linux ~x86-linux"
    IUSE="socks5 +ssl test"
    RESTRICT="!test? ( test )"

    RDEPEND="
        >=dev-python/certifi-2017.4.17[${PYTHON_USEDEP}]
        >=dev-python/chardet-3.0.2[${PYTHON_USEDEP}]
        <dev-python/chardet-4[${PYTHON_USEDEP}]
        >=dev-python/idna-2.5[${PYTHON_USEDEP}]
        <dev-python/idna-3[${PYTHON_USEDEP}]
        <dev-python/urllib3-1.26[${PYTHON_USEDEP}]
        socks5? ( >=dev-python/PySocks-1.5.6[${PYTHON_USEDEP}] )
        ssl? (
            >=dev-python/cryptography-1.3.4[${PYTHON_USEDEP}]
            >=dev-python/pyopenssl-0.14[${PYTHON_USEDEP}]
        )
    "

    BDEPEND="
        dev-python/setuptools[${PYTHON_USEDEP}]
        test? (
            $(python_gen_cond_dep '
                ${RDEPEND}
                dev-python/pytest[${PYTHON_USEDEP}]
                dev-python/pytest-httpbin[${PYTHON_USEDEP}]
                dev-python/pytest-mock[${PYTHON_USEDEP}]
                >=dev-python/PySocks-1.5.6[${PYTHON_USEDEP}]
            ' 'python*')
        )
    "

    python_test() {
        # tests hang with pypy & pypy3
        [[ ${EPYTHON} == pypy* ]] && continue

        epytest
    }

Enforcing implementations is done via putting ``||`` block with all
targets providing the feature in ``REQUIRED_USE``.  The eclass provides
``python_gen_useflags`` function to print valid flag names for specified
implementation list.  Please always use this function instead of listing
actual flag names as it handled phasing implementations out gracefully.
``python_gen_cond_dep`` should also be called with matching target
list.

.. code-block:: bash
   :emphasize-lines: 19,31-33

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7,8} )
    PYTHON_REQ_USE="sqlite"
    inherit distutils-r1

    DESCRIPTION="Toolkit to convert between many translation formats"
    HOMEPAGE="https://github.com/translate/translate"
    SRC_URI="https://github.com/translate/translate/releases/download/${PV}/${P}.tar.gz"

    LICENSE="GPL-2"
    SLOT="0"
    KEYWORDS="amd64 arm64 x86 ~amd64-linux ~x86-linux"
    IUSE="+subtitles"
    REQUIRED_USE="${PYTHON_REQUIRED_USE}
        subtitles? ( || ( $(python_gen_useflags python3_{6,7}) ) )"

    DEPEND=">=dev-python/six-1.10.0[${PYTHON_USEDEP}]"
    RDEPEND="${DEPEND}
        !dev-python/pydiff
        app-text/iso-codes
        >=dev-python/chardet-3.0.4[${PYTHON_USEDEP}]
        >=dev-python/lxml-3.5[${PYTHON_USEDEP}]
        >=dev-python/pycountry-18.5.26[${PYTHON_USEDEP}]
        >=dev-python/python-levenshtein-0.12.0[${PYTHON_USEDEP}]
        sys-devel/gettext
        subtitles? (
            $(python_gen_cond_dep '
                media-video/gaupol[${PYTHON_USEDEP}]
            ' python3_{6,7})
        )
    "

.. index:: python_setup; with implementation parameter
.. index:: DISTUTILS_ALL_SUBPHASE_IMPLS

Restricting interpreters for python_setup
=========================================
A specific case of the restriction described above is when the build
step supports a subset of Python targets for the runtime part.  This
could happen e.g. if package's Python bindings have been ported
to Python 3 but the test suite or building tooling still requires
Python 2.

To support this use case, ``python_setup`` can optionally take a list
of implementations.  This list must be a subset of ``PYTHON_COMPAT``,
and only implementation on the list can be used by ``python_setup``.
Note that you also need to set matching ``REQUIRED_USE``, as otherwise
the function will fail if the user does not enable any of the supported
targets.

.. code-block:: bash
   :emphasize-lines: 19,27

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6

    PYTHON_COMPAT=( python2_7 python3_{5,6,7} )

    inherit python-r1 toolchain-funcs

    DESCRIPTION="Python extension module generator for C and C++ libraries"
    HOMEPAGE="https://www.riverbankcomputing.com/software/sip/intro"
    SRC_URI="https://www.riverbankcomputing.com/static/Downloads/${PN}/${PV}/${P}.tar.gz"

    # Sub-slot based on SIP_API_MAJOR_NR from siplib/sip.h
    SLOT="0/12"
    LICENSE="|| ( GPL-2 GPL-3 SIP )"
    KEYWORDS="alpha amd64 arm arm64 ~hppa ia64 ppc ppc64 ~sparc x86 ~amd64-linux ~x86-linux ~ppc-macos ~x64-macos ~x86-macos"
    REQUIRED_USE="${PYTHON_REQUIRED_USE}
        || ( $(python_gen_useflags 'python2*') )"

    RDEPEND="${PYTHON_DEPS}"
    DEPEND="${RDEPEND}
        sys-devel/bison
        sys-devel/flex

    src_prepare() {
        python_setup 'python2*'
        "${EPYTHON}" build.py prepare || die
        default
    }

    src_configure() {
        configuration() {
            local myconf=(
                "${EPYTHON}"
                "${S}"/configure.py
                --bindir="${EPREFIX}/usr/bin"
                --destdir="$(python_get_sitedir)"
                --incdir="$(python_get_includedir)"
            )
            echo "${myconf[@]}"
            "${myconf[@]}" || die
        }
        python_foreach_impl run_in_build_dir configuration
    }

    src_compile() {
        python_foreach_impl run_in_build_dir default
    }

    src_install() {
        installation() {
            emake DESTDIR="${D}" install
            python_optimize
        }
        python_foreach_impl run_in_build_dir installation

        einstalldocs
    }

The ``distutils-r1`` equivalent of ``python_setup`` parameters is
the ``DISTUTILS_ALL_SUBPHASE_IMPLS`` variable.  Alternatively to global
scope, it can be set in an early phase function (prior to any sub-phase
call).

.. code-block:: bash
   :emphasize-lines: 22,28-30,46

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=5

    PYTHON_COMPAT=(
        pypy
        python3_5 python3_6 python3_7
        python2_7
    )
    PYTHON_REQ_USE='bzip2(+),ssl(+),threads(+)'
    inherit distutils-r1

    DESCRIPTION="Portage is the package management and distribution system for Gentoo"
    HOMEPAGE="https://wiki.gentoo.org/wiki/Project:Portage"
    SRC_URI="mirror://gentoo/${P}.tar.bz2"

    LICENSE="GPL-2"
    KEYWORDS="~alpha ~amd64 ~arm ~arm64 ~hppa ~ia64 ~m68k ~mips ~ppc ~ppc64 ~riscv ~s390 ~sh ~sparc ~x86"
    SLOT="0"
    IUSE="epydoc"
    REQUIRED_USE="epydoc? ( $(python_gen_useflags 'python2*') )"

    DEPEND="
        >=app-arch/tar-1.27
        >=sys-apps/sed-4.0.5 sys-devel/patch
        epydoc? (
            $(python_gen_cond_dep '
                >=dev-python/epydoc-2.0[${PYTHON_USEDEP}]
            ' 'python2*')
        )"
    RDEPEND="
        >=app-arch/tar-1.27
        dev-lang/python-exec:2
        >=sys-apps/sed-4.0.5
        app-shells/bash:0[readline]
        >=app-admin/eselect-1.2
        elibc_glibc? ( >=sys-apps/sandbox-2.2 )
        kernel_linux? ( sys-apps/util-linux )
        >=app-misc/pax-utils-0.1.17"
    PDEPEND="
        >=net-misc/rsync-2.6.4
        userland_GNU? ( >=sys-apps/coreutils-6.4 )"

    pkg_setup() {
        use epydoc && DISTUTILS_ALL_SUBPHASE_IMPLS=( python2.7 )
    }

    python_compile_all() {
        if use epydoc; then
            esetup.py epydoc
        fi
    }

Note that when the restriction is caused by dependencies rather than
package's files, the any-r1 API described below is preferable to this.


.. index:: python_gen_any_dep; python-r1
.. index:: python_check_deps; python-r1

Disjoint build dependencies (any-r1 API)
========================================
Some packages have disjoint sets of runtime and pure build-time
dependencies.  The former need to be built for all enabled
implementations, the latter only for one of them.  The any-r1 API
in ``python-r1`` is specifically suited for expressing that.

Let's consider an example package that uses Sphinx with a plugin
to build documentation.  Naturally, you're going to build the documents
only once, not separately for every enabled target.


Using regular python-r1 API
---------------------------
If you were using the regular API, you'd have to use
``${PYTHON_USEDEP}`` on the dependencies.  The resulting code could look
like the following::

    BDEPEND="
        doc? (
            dev-python/sphinx[${PYTHON_USEDEP}]
            dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]
        )"

    src_compile() {
        ...

        if use doc; then
            python_setup
            emake -C docs html
        fi
    }

If your package is built with support for Python 3.6, 3.7 and 3.8,
then this dependency string will enforce the same targets for Sphinx
and the theme.  However, in practice it will only be used through
Python 3.8.  Normally, this is not such a big deal.

Now imagine your package supports Python 2.7 as well, while Sphinx
does not anymore.  This means that your package will force downgrade
to the old version of ``dev-python/sphinx`` even though it will not
be used via Python 2.7 at all.


Using any-r1 API with python-r1
-------------------------------
As the name suggests, the any-r1 API resembles the API used
by ``python-any-r1`` eclass.  The disjoint build-time dependencies
are declared using ``python_gen_any_dep``, and need to be tested
via ``python_check_deps()`` function.  The presence of the latter
function activates the alternate behavior of ``python_setup``.  Instead
of selecting one of the enabled targets, it will run it to verify
installed dependencies and use one having all dependencies satisfied.

.. code-block:: bash
   :emphasize-lines: 3-6,9-12,18

    BDEPEND="
        doc? (
            $(python_gen_any_dep '
                dev-python/sphinx[${PYTHON_USEDEP}]
                dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]
            ')
        )"

    python_check_deps() {
        python_has_version "dev-python/sphinx[${PYTHON_USEDEP}]" &&
        python_has_version "dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]"
    }

    src_compile() {
        ...

        if use doc; then
            python_setup
            emake -C docs html
        fi
    }

Note that ``python_setup`` may select an implementation that is not even
enabled via ``PYTHON_TARGETS``.  The goal is to try hard to avoid
requiring user to change USE flags on dependencies if possible.

An interesting side effect of that is that the supported targets
in the dependencies can be a subset of the one in package.  For example,
we have used this API to add Python 3.8 support to packages before
``dev-python/sphinx`` supported it — the eclass implicitly forced using
another implementation for Sphinx.


Different sets of build-time dependencies
-----------------------------------------
Let's consider the case when Python is used at build-time for something
else still.  In that case, we want ``python_setup`` to work
unconditionally but enforce dependencies only with ``doc`` flag enabled.

.. code-block:: bash
   :emphasize-lines: 9-13,16

    BDEPEND="
        doc? (
            $(python_gen_any_dep '
                dev-python/sphinx[${PYTHON_USEDEP}]
                dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]
            ')
        )"

    python_check_deps() {
        use doc || return 0
        python_has_version "dev-python/sphinx[${PYTHON_USEDEP}]" &&
        python_has_version "dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]"
    }

    src_compile() {
        python_setup

        ...

        use doc && emake -C docs html
    }

Note that ``python_setup`` behaves according to the any-r1 API here.
While it will not enforce doc dependencies with ``doc`` flag disabled,
it will use *any* interpreter that is supported and installed, even
if it is not enabled explicitly in ``PYTHON_TARGETS``.


Using any-r1 API with distutils-r1
----------------------------------
The alternate build dependency API also integrates with ``distutils-r1``
eclass.  If ``python_check_deps()`` is declared, the ``python_*_all()``
sub-phase functions are called with the interpreter selected according
to any-r1 rules.

.. code-block:: bash
   :emphasize-lines: 3-6,9-13

    BDEPEND="
        doc? (
            $(python_gen_any_dep '
                dev-python/sphinx[${PYTHON_USEDEP}]
                dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]
            ')
        )"

    python_check_deps() {
        use doc || return 0
        python_has_version "dev-python/sphinx[${PYTHON_USEDEP}]" &&
        python_has_version "dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]"
    }

    python_compile_all() {
        use doc && emake -C docs html
    }

Note that ``distutils-r1`` calls ``python_setup`` unconditionally,
therefore ``python_check_deps()`` needs to account for that.

Normally you won't have to use this API for Sphinx though —
``distutils_enable_sphinx`` does precisely that for you.


Combining any-r1 API with implementation restrictions
=====================================================
Both APIs described above can be combined.  This can be used when
build-time scripts support a subset of implementations supported
by the package itself, and by its build-time dependencies.  For example,
if the package uses ``dev-util/scons`` build system with ``SConstruct``
files using Python 2 construct.

There are two approaches to achieve that: either the build-time
implementation list needs to be passed to ``python_setup``,
or ``python_check_deps`` needs to explicitly reject unsupported targets.
In both cases, a matching implementation list needs to be passed
to ``python_gen_any_dep``.

.. code-block:: bash
   :emphasize-lines: 25,28-30,46

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 python3_6 )
    inherit python-r1 toolchain-funcs

    DESCRIPTION="GPS daemon and library for USB/serial GPS devices and GPS/mapping clients"
    HOMEPAGE="https://gpsd.gitlab.io/gpsd/"
    SRC_URI="mirror://nongnu/${PN}/${P}.tar.gz"

    LICENSE="BSD"
    SLOT="0/24"
    KEYWORDS="~amd64 ~arm ~ppc ~ppc64 ~sparc ~x86"

    IUSE="python"
    REQUIRED_USE="
        python? ( ${PYTHON_REQUIRED_USE} )"

    RDEPEND="
        >=net-misc/pps-tools-0.0.20120407
        python? ( ${PYTHON_DEPS} )"
    DEPEND="${RDEPEND}
        $(python_gen_any_dep '>=dev-util/scons-2.3.0[${PYTHON_USEDEP}]' -2)
        virtual/pkgconfig"

    python_check_deps() {
        python_has_version ">=dev-util/scons-2.3.0[${PYTHON_USEDEP}]"
    }

    src_configure() {
        myesconsargs=(
            prefix="${EPREFIX}/usr"
            libdir="\$prefix/$(get_libdir)"
            udevdir="$(get_udevdir)"
            chrpath=False
            gpsd_user=gpsd
            gpsd_group=uucp
            nostrip=True
            manbuild=False
            $(use_scons python)
        )

        # SConstruct uses py2 constructs
        python_setup -2
    }

.. code-block:: bash
   :emphasize-lines: 25,28-31,46

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 python3_6 )
    inherit python-r1 toolchain-funcs

    DESCRIPTION="GPS daemon and library for USB/serial GPS devices and GPS/mapping clients"
    HOMEPAGE="https://gpsd.gitlab.io/gpsd/"
    SRC_URI="mirror://nongnu/${PN}/${P}.tar.gz"

    LICENSE="BSD"
    SLOT="0/24"
    KEYWORDS="~amd64 ~arm ~ppc ~ppc64 ~sparc ~x86"

    IUSE="python"
    REQUIRED_USE="
        python? ( ${PYTHON_REQUIRED_USE} )"

    RDEPEND="
        >=net-misc/pps-tools-0.0.20120407
        python? ( ${PYTHON_DEPS} )"
    DEPEND="${RDEPEND}
        $(python_gen_any_dep '>=dev-util/scons-2.3.0[${PYTHON_USEDEP}]' -2)
        virtual/pkgconfig"

    python_check_deps() {
        python_is_python3 && return 1
        python_has_version ">=dev-util/scons-2.3.0[${PYTHON_USEDEP}]"
    }

    src_configure() {
        myesconsargs=(
            prefix="${EPREFIX}/usr"
            libdir="\$prefix/$(get_libdir)"
            udevdir="$(get_udevdir)"
            chrpath=False
            gpsd_user=gpsd
            gpsd_group=uucp
            nostrip=True
            manbuild=False
            $(use_scons python)
        )

        python_setup
    }
