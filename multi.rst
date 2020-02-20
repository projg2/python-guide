===============================
python-r1 — multi-impl packages
===============================

.. highlight:: bash

The ``python-r1`` eclass is used to install multi-impl packages.
It is considered an expert eclass — when possible, you should prefer
using ``python-single-r1`` instead.  For packages using distutils
or a similar Python build system, ``distutils-r1`` eclass should be used
instead.


Manual install
==============
The simplest case of multi-impl package is a package without a specific
build system.  The modules need to be installed manually here,
and ``python_foreach_impl`` function is used to repeat the install step
for all enabled implementations.

For simple use cases, the install command can be inlined:

.. code-block:: bash
   :emphasize-lines: 6,7,17,19,23

    # Copyright 1999-2020 Gentoo Foundation
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 )
    inherit python-r1

    DESCRIPTION="MySQL abstraction layer for python"
    HOMEPAGE="http://software.fionet.com/pSQL/"
    SRC_URI="http://software.fionet.com/pSQL/release/${P}.tar.gz"

    SLOT="0"
    LICENSE="GPL-2"
    KEYWORDS="~amd64 ~x86"
    IUSE=""
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    RDEPEND="${PYTHON_DEPS}"
    DEPEND="${RDEPEND}"

    src_install() {
        python_foreach_impl python_domodule pSQL.py
    }

While ``python_foreach_impl`` can be repeated multiple times, it is
generally better to declare a function when multiple install commands
need to be executed:

.. code-block:: bash
   :emphasize-lines: 22-25,28

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 )
    inherit python-r1

    DESCRIPTION="Proxy cache for Gentoo packages"
    HOMEPAGE="https://sourceforge.net/projects/http-replicator"
    SRC_URI="mirror://sourceforge/${PN}/${P}.tgz"

    LICENSE="GPL-2"
    SLOT="0"
    KEYWORDS="~alpha amd64 hppa ppc ~sparc x86"
    IUSE=""
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    RDEPEND="${PYTHON_DEPS}"
    DEPEND="${RDEPEND}"

    python_install() {
        python_doscript http-replicator
        python_domodule *.py
    }

    src_install() {
        python_foreach_impl python_install
    }


Dependencies
============
When depending on other Python packages, USE dependencies need to be
declared in order to ensure that the dependencies would be built against
all the Python implementations enabled for the package.  This is easily
done via appending the USE dependency string from ``${PYTHON_USEDEP}``
to the dependencies::

    RDEPEND="${PYTHON_DEPS}
        sys-apps/portage[${PYTHON_USEDEP}]
    "
    DEPEND="${RDEPEND}"


Pure Python autotools package
=============================
Another typical case for this eclass is to handle a pure Python package
with a non-standard build system.  In this case, it is generally
necessary to call phase functions via ``python_foreach_impl``.  Whenever
possible, out-of-source builds are recommended (i.e. installing to
separate directories from a single source directory).


.. code-block:: bash
   :emphasize-lines: 32,36,40,44

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI="6"
    PYTHON_COMPAT=( python3_6 )

    inherit autotools python-r1

    DESCRIPTION="Python wrapper for libcangjie"
    HOMEPAGE="http://cangjians.github.io/"
    SRC_URI="https://github.com/Cangjians/py${PN}/releases/download/v${PV}/${P#py}.tar.xz"

    LICENSE="LGPL-3+"
    SLOT="0"
    KEYWORDS="~amd64 ~x86"
    IUSE=""
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    RDEPEND="${PYTHON_DEPS}
        app-i18n/libcangjie"
    DEPEND="${RDEPEND}
        dev-python/cython[${PYTHON_USEDEP}]
        virtual/pkgconfig"

    src_prepare() {
        default
        eautoreconf
    }

    src_configure() {
        local ECONF_SOURCE=${S}
        python_foreach_impl run_in_build_dir default
    }

    src_compile() {
        python_foreach_impl run_in_build_dir default
    }

    src_test() {
        python_foreach_impl run_in_build_dir default
    }

    src_install() {
        python_foreach_impl run_in_build_dir default
        einstalldocs
        find "${D}" -name '*.la' -delete || die
    }

Note the use of ``run_in_build_dir`` helper from ``multibuild`` eclass
(direct inherit is unnecessary here, as it is considered implicit part
of ``python-r1`` API).  It changes the directory to ``BUILD_DIR`` (which
is set by ``python_foreach_impl`` to a unique directory for each
implementation) and runs the specified command there.  In this case,
the ebuild performs autotools out-of-source build in a dedicated
directory for every interpreter enabled.

Also note that the in-build-dir call to ``default`` does not install
documentation from source directory, hence the additional
``einstalldocs`` call.  Libtool-based packages install ``.la`` files
that are unnecessary for Python extensions, hence they are removed
afterwards.

If the package in question does not support out-of-source builds
(e.g. due to a buggy build system), ``python_copy_sources`` function
can be used to duplicate the package's sources in build directories
for each implementation.  The same ebuild easily can be changed
to do that:

.. code-block:: bash
   :emphasize-lines: 28,32,36,40,44

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI="6"
    PYTHON_COMPAT=( python3_6 )

    inherit autotools python-r1

    DESCRIPTION="Python wrapper for libcangjie"
    HOMEPAGE="http://cangjians.github.io/"
    SRC_URI="https://github.com/Cangjians/py${PN}/releases/download/v${PV}/${P#py}.tar.xz"

    LICENSE="LGPL-3+"
    SLOT="0"
    KEYWORDS="~amd64 ~x86"
    IUSE=""
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    RDEPEND="${PYTHON_DEPS}
        app-i18n/libcangjie"
    DEPEND="${RDEPEND}
        dev-python/cython[${PYTHON_USEDEP}]
        virtual/pkgconfig"

    src_prepare() {
        default
        eautoreconf
        python_copy_sources
    }

    src_configure() {
        python_foreach_impl run_in_build_dir default
    }

    src_compile() {
        python_foreach_impl run_in_build_dir default
    }

    src_test() {
        python_foreach_impl run_in_build_dir default
    }

    src_install() {
        python_foreach_impl run_in_build_dir default
        einstalldocs
        find "${D}" -name '*.la' -delete || die
    }

Note that besides adding ``python_copy_sources`` call, ``ECONF_SOURCE``
has been removed in order to disable out-of-source builds.


Conditional Python use
======================
When the package installs Python components conditionally to a USE flag,
the respective USE conditional needs to be consistently used in metadata
variables and in ``python_foreach_impl`` calls.

.. code-block:: bash
   :emphasize-lines: 15,16,20-22,42-48

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6
    PYTHON_COMPAT=( python2_7 )

    inherit gnome2 python-r1

    DESCRIPTION="Canvas widget for GTK+ using the cairo 2D library for drawing"
    HOMEPAGE="https://wiki.gnome.org/GooCanvas"

    LICENSE="LGPL-2"
    SLOT="2.0"
    KEYWORDS="~alpha amd64 ia64 ppc ppc64 sparc x86"
    IUSE="python"
    REQUIRED_USE="python? ( ${PYTHON_REQUIRED_USE} )"

    # python only enables python specific binding override
    RDEPEND="
        python? (
            ${PYTHON_DEPS}
            >=dev-python/pygobject-2.90.4:3[${PYTHON_USEDEP}] )
    "
    DEPEND="${RDEPEND}"

    src_prepare() {
        # Python bindings are built/installed manually.
        sed -e "/SUBDIRS = python/d" -i bindings/Makefile.am \
            bindings/Makefile.in || die

        gnome2_src_prepare
    }

    src_configure() {
        gnome2_src_configure \
            --disable-python
    }

    src_install() {
        gnome2_src_install

        if use python; then
            sub_install() {
                python_moduleinto $(python -c "import gi;print gi._overridesdir")
                python_domodule bindings/python/GooCanvas.py
            }
            python_foreach_impl sub_install
        fi
    }

Note that in many cases, you will end up having to disable upstream
rules for installing Python files as they are suitable only for
single-impl installs.


Additional build-time Python use
================================
Some packages additionally require Python at build time, independently
of Python components installed (i.e. outside ``python_foreach_impl``).
The eclass provides extensive API for this purpose but for now we'll
focus on the simplest case where the global code does not have any
dependencies or they are a subset of dependencies declared already.

In this case, it is sufficient to call ``python_setup`` before
the routine requiring Python.  It will choose the most preferred
of enabled implementations, and set the global environment for it.  Note
that it is entirely normal that the same environment will be set inside
``python_foreach_impl`` afterwards.

.. code-block:: bash
   :linenos:
   :emphasize-lines: 17,18,20,21,24,28-34,38-40

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI="7"

    PYTHON_COMPAT=( python{3_6,3_7} )
    PYTHON_REQ_USE="ncurses,readline"
    inherit python-r1

    DESCRIPTION="QEMU + Kernel-based Virtual Machine userland tools"
    HOMEPAGE="http://www.qemu.org http://www.linux-kvm.org"
    SRC_URI="http://wiki.qemu-project.org/download/${P}.tar.xz"

    LICENSE="GPL-2 LGPL-2 BSD-2"
    SLOT="0"
    KEYWORDS="amd64 ~arm64 ~ppc ~ppc64 x86"
    IUSE="python"
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    BDEPEND="${PYTHON_DEPS}"
    RDEPEND="python? ( ${PYTHON_DEPS} )"

    src_configure() {
        python_setup
        ./configure || die
    }

    qemu_python_install() {
        python_domodule "${S}/python/qemu"

        python_doscript "${S}/scripts/kvm/vmxcap"
        python_doscript "${S}/scripts/qmp/qmp-shell"
        python_doscript "${S}/scripts/qmp/qemu-ga-client"
    }

    src_install() {
        default
        if use python; then
            python_foreach_impl qemu_python_install
        fi
    }

Note that the parts affecting installation of runtime components
(``RDEPEND``, ``python_foreach_impl``) are made conditional to the USE
flag, while parts affecting build time (``REQUIRED_USE``, ``BDEPEND``,
``python_setup``) are unconditional.
