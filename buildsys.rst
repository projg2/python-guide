================================================
Integration with build systems written in Python
================================================

Besides the build systems commonly used by Python packages there are
a few build systems written in Python and requiring the packages
to script them in Python.  This makes it necessary to use a proper
Python eclass in order to declare their compatibility with Python
versions.


Meson
=====
Meson_ build system is a fairly recent addition.  While it is written
in Python, its scripts use a custom Python-agnostic script language.
Hence, it can be treated as an arbitrary external tool and does not need
any Python eclass.


SCons
=====
SCons_ has gained Python 3 support quite recently.  At the same time,
many old script files were written for Python 2 and fail when run
via Python 3 SCons.  For this reason, it is necessary to use Python
eclass when using SCons.

In the simplest case when the package has no other use for Python,
it is sufficient to declare ``PYTHON_COMPAT`` for the SCons scripts
in the package, and inherit ``python-any-r1`` prior to ``scons-utils``
(this happens naturally when you sort includes by eclass name).
The latter eclass takes care of setting as much as possible.

If the package has other Python components, it is necessary to account
for them and use an appropriate eclass as detailed in the eclass choice
chapter.


Build-time use with no extra dependencies
-----------------------------------------
If the package either has no other Python components than SCons, or all
of them are purely build-time and have no dependencies, it is sufficient
to inherit ``python-any-r1``.  The eclass takes care of setting
``BDEPEND`` along with matching ``python_check_deps()``.

.. code-block:: bash
   :emphasize-lines: 6,7

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7} )
    inherit python-any-r1 scons-utils toolchain-funcs

    COMMIT="6e5e8a57628095d8d0c8bbb38187afb0f3a42112"
    DESCRIPTION="Userspace Xbox 360 Controller driver"
    HOMEPAGE="https://xboxdrv.gitlab.io"
    SRC_URI="https://github.com/chewi/xboxdrv/archive/${COMMIT}.tar.gz -> ${P}.tar.gz"
    S="${WORKDIR}/${PN}-${COMMIT}"

    LICENSE="GPL-3"
    SLOT="0"
    KEYWORDS="~amd64 ~x86"

    RDEPEND="
        dev-libs/boost:=
        dev-libs/dbus-glib
        dev-libs/glib:2
        sys-apps/dbus
        virtual/libudev:=
        virtual/libusb:1
        x11-libs/libX11
    "

    DEPEND="
        ${RDEPEND}
    "

    BDEPEND="
        dev-util/glib-utils
        virtual/pkgconfig
    "

    src_compile() {
        escons \
            BUILD=custom \
            CXX="$(tc-getCXX)" \
            AR="$(tc-getAR)" \
            RANLIB="$(tc-getRANLIB)" \
            CXXFLAGS="-Wall ${CXXFLAGS}" \
            LINKFLAGS="${LDFLAGS}"
    }

    src_install() {
        dobin xboxdrv
        doman doc/xboxdrv.1
        dodoc AUTHORS NEWS PROTOCOL README.md TODO
    }


Build-time use with extra dependencies
--------------------------------------
If the package has extra dependencies, you need to take care of *all*
dependencies yourself.  This is because ``python_gen_any_dep`` cannot
be combined.

.. code-block:: bash
   :emphasize-lines: 6,7,35,51

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7} )
    inherit python-any-r1 scons-utils toolchain-funcs

    MY_P=${PN}-src-r${PV/_rc/-rc}
    DESCRIPTION="A high-performance, open source, schema-free document-oriented database"
    HOMEPAGE="https://www.mongodb.com"
    SRC_URI="https://fastdl.mongodb.org/src/${MY_P}.tar.gz"
    S="${WORKDIR}/${MY_P}"

    LICENSE="Apache-2.0 SSPL-1"
    SLOT="0"
    KEYWORDS="~amd64"
    IUSE="test +tools"
    RESTRICT="!test? ( test )"

    RDEPEND="acct-group/mongodb
        acct-user/mongodb
        >=app-arch/snappy-1.1.3
        >=dev-cpp/yaml-cpp-0.6.2:=
        >=dev-libs/boost-1.70:=[threads(+)]
        >=dev-libs/libpcre-8.42[cxx]
        app-arch/zstd
        dev-libs/snowball-stemmer
        net-libs/libpcap
        >=sys-libs/zlib-1.2.11:="
    DEPEND="${RDEPEND}
        ${PYTHON_DEPS}
        $(python_gen_any_dep '
            test? ( dev-python/pymongo[${PYTHON_USEDEP}] )
            >=dev-util/scons-2.5.0[${PYTHON_USEDEP}]
            dev-python/cheetah3[${PYTHON_USEDEP}]
            dev-python/psutil[${PYTHON_USEDEP}]
            dev-python/pyyaml[${PYTHON_USEDEP}]
            virtual/python-typing[${PYTHON_USEDEP}]
        ')
        sys-libs/ncurses:0=
        sys-libs/readline:0="
    PDEPEND="tools? ( >=app-admin/mongo-tools-${PV} )"

    python_check_deps() {
        if use test; then
            python_has_version "dev-python/pymongo[${PYTHON_USEDEP}]" ||
                return 1
        fi

        python_has_version ">=dev-util/scons-2.5.0[${PYTHON_USEDEP}]" &&
        python_has_version "dev-python/cheetah3[${PYTHON_USEDEP}]" &&
        python_has_version "dev-python/psutil[${PYTHON_USEDEP}]" &&
        python_has_version "dev-python/pyyaml[${PYTHON_USEDEP}]" &&
        python_has_version "virtual/python-typing[${PYTHON_USEDEP}]"
    }

    src_configure() {
        scons_opts=(
            CC="$(tc-getCC)"
            CXX="$(tc-getCXX)"

            --disable-warnings-as-errors
            --use-system-boost
            --use-system-pcre
            --use-system-snappy
            --use-system-stemmer
            --use-system-yaml
            --use-system-zlib
            --use-system-zstd
        )

        default
    }

    src_compile() {
        escons "${scons_opts[@]}" core tools
    }

    src_test() {
        "${EPYTHON}" ./buildscripts/resmoke.py --dbpathPrefix=test \
            --suites core --jobs=$(makeopts_jobs) || die "Tests failed"
    }

    src_install() {
        escons "${scons_opts[@]}" --nostrip install --prefix="${ED}"/usr
    }


Single-impl package
-------------------
If the package needs to install some Python components, and single-impl
install is appropriate, you need to combine ``python-single-r1``
with ``scons-utils``.  In this case, the eclass takes care of everything
needed for SCons, and you take care of everything needed for your
package.

.. code-block:: bash
   :emphasize-lines: 6,7,18

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=5

    PYTHON_COMPAT=( python2_7 )
    inherit eutils python-single-r1 scons-utils toolchain-funcs

    DESCRIPTION="Molecular dynamics by NMR data analysis"
    HOMEPAGE="https://www.nmr-relax.com/"
    SRC_URI="http://download.gna.org/relax/${P}.src.tar.bz2"

    SLOT="0"
    LICENSE="GPL-2"
    KEYWORDS="~amd64 ~x86 ~amd64-linux ~x86-linux"
    IUSE="test"
    RESTRICT="!test? ( test )"
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    RDEPEND="
        ${PYTHON_DEPS}
        $(python_gen_cond_dep "
            dev-python/Numdifftools[\${PYTHON_USEDEP}]
            || (
                dev-python/matplotlib-python2[\${PYTHON_USEDEP}]
                dev-python/matplotlib[\${PYTHON_USEDEP}]
            )
            || (
                dev-python/numpy-python2[\${PYTHON_USEDEP}]
                dev-python/numpy[\${PYTHON_USEDEP}]
            )
            dev-python/wxpython:${WX_GTK_VER}[\${PYTHON_USEDEP}]
            sci-chemistry/pymol[\${PYTHON_USEDEP}]
            >=sci-libs/bmrblib-1.0.3[\${PYTHON_USEDEP}]
            >=sci-libs/minfx-1.0.11[\${PYTHON_USEDEP}]
            || (
                sci-libs/scipy-python2[\${PYTHON_USEDEP}]
                sci-libs/scipy[\${PYTHON_USEDEP}]
            )
        ")
        sci-chemistry/molmol
        sci-chemistry/vmd
        sci-visualization/grace
        sci-visualization/opendx"
    DEPEND="${RDEPEND}
        media-gfx/pngcrush
        test? ( ${RDEPEND} )
        "

    src_compile() {
        tc-export CC
        escons
    }

    src_install() {
        python_moduleinto ${PN}
        python_domodule *

        make_wrapper ${PN}-nmr "${EPYTHON} $(python_get_sitedir)/${PN}/${PN}.py $@"
    }


Single-impl package with conditional Python install
---------------------------------------------------
If the runtime part of the package uses Python only conditionally,
the use is similar to a package with unconditional build-time
and conditional runtime dependency on Python.  That is, build-time
dependencies, ``REQUIRED_USE`` and ``pkg_setup`` must be called
unconditionally.

.. code-block:: bash
   :emphasize-lines: 6,11,23,50

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7,8} )

    FORTRAN_NEEDED=fortran
    FORTRAN_STANDARD=90

    inherit python-single-r1 scons-utils toolchain-funcs

    DESCRIPTION="Object-oriented tool suite for chemical kinetics, thermodynamics, and transport"
    HOMEPAGE="https://www.cantera.org"
    SRC_URI="https://github.com/Cantera/${PN}/archive/v${PV}.tar.gz -> ${P}.tar.gz"

    LICENSE="BSD"
    SLOT="0"
    KEYWORDS="amd64 ~x86"
    IUSE="fortran pch +python"

    REQUIRED_USE="
        ${PYTHON_REQUIRED_USE}
    "

    RDEPEND="
        python? (
            ${PYTHON_DEPS}
            $(python_gen_cond_dep '
                dev-python/numpy[${PYTHON_USEDEP}]
            ')
        )
        <sci-libs/sundials-5.1.0:0=
    "

    DEPEND="
        ${RDEPEND}
        dev-cpp/eigen:3
        dev-libs/boost
        dev-libs/libfmt
        python? (
            $(python_gen_cond_dep '
                dev-python/cython[${PYTHON_USEDEP}]
            ')
        )
    "

    pkg_setup() {
        fortran-2_pkg_setup
        python-single-r1_pkg_setup
    }

    src_configure() {
        scons_vars=(
            CC="$(tc-getCC)"
            CXX="$(tc-getCXX)"
            cc_flags="${CXXFLAGS}"
            cxx_flags="-std=c++11"
            debug="no"
            FORTRAN="$(tc-getFC)"
            FORTRANFLAGS="${CXXFLAGS}"
            optimize_flags="-Wno-inline"
            renamed_shared_libraries="no"
            use_pch=$(usex pch)
            system_fmt="y"
            system_sundials="y"
            system_eigen="y"
            env_vars="all"
            extra_inc_dirs="/usr/include/eigen3"
        )

        scons_targets=(
            f90_interface=$(usex fortran y n)
            python2_package="none"
        )

        if use python ; then
            scons_targets+=( python3_package="full" python3_cmd="${EPYTHON}" )
        else
            scons_targets+=( python3_package="none" )
        fi
    }

    src_compile() {
        escons build "${scons_vars[@]}" "${scons_targets[@]}" prefix="/usr"
    }

    src_test() {
        escons test
    }

    src_install() {
        escons install stage_dir="${D}" libdirname="$(get_libdir)"
        python_optimize
    }


Pure Python multi-impl package
------------------------------
When you are dealing with a pure Python package using SCons, it makes
sense to use plain ``python-r1`` API.  This means that SCons is going
to be called from a ``python_foreach_impl`` loop only.

.. code-block:: bash
   :emphasize-lines: 6,7,17,29,57,62

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 )
    inherit fortran-2 python-r1 scons-utils toolchain-funcs

    DESCRIPTION="Automated pipeline for performing Poisson-Boltzmann electrostatics calculations"
    HOMEPAGE="https://www.poissonboltzmann.org/"
    SRC_URI="https://github.com/Electrostatics/apbs-${PN}/releases/download/${P}/${PN}-src-${PV}.tar.gz"

    SLOT="0"
    LICENSE="BSD"
    KEYWORDS="amd64 x86 ~amd64-linux ~x86-linux"
    IUSE="opal"
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    RDEPEND="${PYTHON_DEPS}
        || (
            dev-python/numpy-python2[${PYTHON_USEDEP}]
            dev-python/numpy[${PYTHON_USEDEP}]
        )
        sci-chemistry/openbabel-python[${PYTHON_USEDEP}]
        opal? ( dev-python/zsi[${PYTHON_USEDEP}] )
        "
    DEPEND="${RDEPEND}
        dev-lang/swig:0
        dev-util/scons[${PYTHON_USEDEP}]"

    src_prepare() {
        find -type f \( -name "*.pyc" -o -name "*.pyo" \) -delete || die

        eapply "${PATCHES[@]}"
        eapply_user
        rm -rf scons || die

        python_copy_sources
    }

    python_configure() {
        tc-export CXX
        cat > "${BUILD_DIR}"/build_config.py <<-EOF || die
            PREFIX="${D}/$(python_get_sitedir)/${PN}"
            APBS="${EPREFIX}/usr/bin/apbs"
            MAX_ATOMS=10000
            BUILD_PDB2PKA=False
            REBUILD_SWIG=True
        EOF
    }

    src_configure() {
        python_foreach_impl python_configure
    }

    src_compile() {
        python_foreach_impl run_in_build_dir escons
    }

    python_install() {
        cd "${BUILD_DIR}" || die
        escons install
        python_optimize
    }

    src_install() {
        python_foreach_impl python_install
    }


Hybrid python-r1 + SCons package
--------------------------------
Finally, let's consider a package that uses SCons as a build system
and installs Python components independently of it.  This could be
e.g. a C/C++ program with separate Python bindings.

Let's presume that the Python bindings need to be installed manually,
and they support a wider target range than the build system.  In this
case, the any-r1 API is recommended.

.. code-block:: bash
   :emphasize-lines: 25,28-30,46

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 python3_6 )
    inherit python-r1 scons-utils toolchain-funcs

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

    src_compile() {
        export CHRPATH=
        tc-export CC CXX PKG_CONFIG
        export SHLINKFLAGS=${LDFLAGS} LINKFLAGS=${LDFLAGS}
        escons
    }

    src_install() {
        DESTDIR="${D}" escons install
        use python && python_foreach_impl python_domodule gps
    }


waf
===
The waf_ build system is written in Python and bundled with the packages
using it.  Therefore, it is necessary to combine ``waf-utils`` eclass
with one of the Python eclasses.

Since SCons does not have any dependencies beside the Python
interpreter, the integration is generally simple.  You consider waf
like any other build-time script, and use the eclass implied by other
Python components in package.

Furthermore, since waf requires threading support in the Python
interpreter, it is necessary to add ``PYTHON_REQ_USE='threads(+)'`` in
all waf packages (combined with individual package requirements if
applicable).


Build-time use
--------------
If waf is the only build-time Python script in the package, it is only
necessary to add ``PYTHON_REQ_USE`` and ``${PYTHON_DEPS}`` to build-time
dependencies.  If the package had other Python dependencies, you would
specify them instead.

.. code-block:: bash
   :emphasize-lines: 6,7,10,22

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 )
    PYTHON_REQ_USE='threads(+)'
    NO_WAF_LIBDIR=yes

    inherit python-any-r1 waf-utils

    DESCRIPTION="C++ Template Unit Test Framework"
    HOMEPAGE="http://mrzechonek.github.io/tut-framework/"
    SRC_URI="https://github.com/mrzechonek/tut-framework/archive/${PV//./-}.tar.gz -> ${P}.tar.gz"
    S="${WORKDIR}/tut-framework-${PV//./-}"

    LICENSE="BSD-2"
    SLOT="0"
    KEYWORDS="~amd64 ~x86"
    IUSE=""

    BDEPEND=${PYTHON_DEPS}


Single-impl package
-------------------
The rules for integrating simple-impl package are roughly the same
as for pure ``python-single-r1`` use.  Again, waf requires only plain
build-time ``${PYTHON_DEPS}`` and ``PYTHON_REQ_USE``.

.. code-block:: bash
   :emphasize-lines: 5,6,8,31,42

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6
    PYTHON_COMPAT=( python2_7 )
    PYTHON_REQ_USE="threads"

    inherit python-single-r1 waf-utils

    DESCRIPTION="An LDAP-like embedded database"
    HOMEPAGE="https://ldb.samba.org/"
    SRC_URI="https://www.samba.org/ftp/pub/${PN}/${P}.tar.gz"

    LICENSE="LGPL-3"
    SLOT="0/${PV}"
    KEYWORDS="~alpha amd64 arm ~arm64 ~hppa ia64 ~mips ppc ppc64 ~s390 ~sh sparc x86"
    IUSE="+ldap python"
    REQUIRED_USE="python? ( ${PYTHON_REQUIRED_USE} )"

    RDEPEND="!elibc_FreeBSD? ( dev-libs/libbsd )
        dev-libs/popt
        >=sys-libs/talloc-2.1.8[python?]
        >=sys-libs/tevent-0.9.31[python(+)?]
        >=sys-libs/tdb-1.3.12[python?]
        python? ( ${PYTHON_DEPS} )
        ldap? ( net-nds/openldap )
        "

    DEPEND="dev-libs/libxslt
        virtual/pkgconfig
        ${PYTHON_DEPS}
        ${RDEPEND}"

    WAF_BINARY="${S}/buildtools/bin/waf"

    PATCHES=(
        "${FILESDIR}"/${PN}-1.1.27-optional_packages.patch
        "${FILESDIR}"/${P}-disable-python.patch
    )

    pkg_setup() {
        python-single-r1_pkg_setup
    }

    src_configure() {
        local myconf=(
            $(usex ldap '' --disable-ldap)
            $(usex python '' '--disable-python')
            --disable-rpath
            --disable-rpath-install --bundled-libraries=NONE
            --with-modulesdir="${EPREFIX}"/usr/$(get_libdir)/samba
            --builtin-libraries=NONE
        )
        waf-utils_src_configure "${myconf[@]}"
    }


.. _Meson: https://mesonbuild.com/

.. _SCons: https://scons.org/

.. _waf: https://waf.io/
