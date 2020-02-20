=======================================
python-single-r1 — single-impl packages
=======================================

.. highlight:: bash

The ``python-single-r1`` eclass is used to install single-impl packages.
It is probably the easiest eclass to use, and it is recommended over
``python-r1`` whenever multi-impl support would add unnecessary
complexity.  However, for packages using distutils or a similar Python
build system, ``distutils-r1`` eclass should be used instead.


Basic use for unconditional Python
==================================
The defining feature of this eclass is that it defines a ``pkg_setup``
phase.  It normally calls ``python_setup`` function in order
to determine the interpreter selected by user, and set the global
environment appropriately.

This means that a most trivial package using an autotools-compatible
build system along with unconditional dependency on Python could look
like the following:

.. code-block:: bash
   :emphasize-lines: 6,7,17,20

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python2_7 )
    inherit python-single-r1

    DESCRIPTION="Scripts to prepare and plot VOACAP propagation predictions"
    HOMEPAGE="https://www.qsl.net/h/hz1jw//pythonprop/"
    SRC_URI="mirror://sourceforge/${PN}/${P}.tar.gz"

    LICENSE="GPL-2+"
    SLOT="0"
    KEYWORDS="~amd64 ~x86"
    IUSE=""
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    RDEPEND="
        ${PYTHON_DEPS}
        ...
    "
    BDEPEND="${RDEPEND}"

This ebuild demonstrates the absolute minimum working code.  Only
the four highlighted lines are specific to Python eclasses, plus
the implicitly exported ``pkg_setup`` phase.


Dependencies
============
When depending on other Python packages, USE dependencies need to be
declared in order to ensure that the dependencies would be built against
the Python implementation used for the package.  The exact dependency
string depends on whether the target package is single-impl
or multi-impl.

When depending on other single-impl packages, the eclass-defined
``${PYTHON_SINGLE_USEDEP}`` variable can be used to inject the correct
USE dependency::

    RDEPEND="
        ...
        dev-python/basemap[${PYTHON_SINGLE_USEDEP}]
    "

When depending on multi-impl packages, a more complex construct must
be used.  The ``python_gen_cond_dep`` generator function is used
to copy the specified dependency template for all supported
implementations, and substitute ``${PYTHON_MULTI_USEDEP}`` template
inside it::

    RDEPEND="
        ...
        $(python_gen_cond_dep '
            dev-python/matplotlib-python2[gtk2,${PYTHON_MULTI_USEDEP}]
        ')
    "

Please note that in this context, ``${...}`` is used as a literal
template substitution key, so it must be escaped to prevent bash from
substituting it immediately.  In the above example, single quotes
are used for this purpose.  When other variables are used, double quotes
with explicit escapes have to be used::

    RDEPEND="
        ...
        $(python_gen_cond_dep "
            dev-python/wxpython:${WX_GTK_VER}[\${PYTHON_MULTI_USEDEP}]
        ")"

As demonstrated above, the USE dependency string can be combined with
other USE dependencies.  ``PYTHON_SINGLE_USEDEP`` can be used both
inside and outside ``python_gen_cond_dep``, while
``PYTHON_MULTI_USEDEP`` only inside it.


Conditional Python use
======================
The examples so far assumed that Python is used unconditionally.
If Python support is conditional to a USE flag, appropriate USE
conditionals need to be used in metadata variables, and ``pkg_setup``
needs to be rewritten to call the default implementation conditionally:

.. code-block:: bash
   :emphasize-lines: 16,17,20,21,23-27,30,35

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6

    PYTHON_COMPAT=( python2_7 )
    inherit python-single-r1

    DESCRIPTION="Yet more Objects for (High Energy Physics) Data Analysis"
    HOMEPAGE="http://yoda.hepforge.org/"
    SRC_URI="http://www.hepforge.org/archive/${PN}/${P}.tar.bz2"

    LICENSE="GPL-2"
    SLOT="0/${PV}"
    KEYWORDS="~amd64 ~x86 ~amd64-linux ~x86-linux"
    IUSE="python root"
    REQUIRED_USE="python? ( ${PYTHON_REQUIRED_USE} )"

    RDEPEND="
        python? ( ${PYTHON_DEPS} )
        root? ( sci-physics/root:=[python=,${PYTHON_SINGLE_USEDEP}] )"
    DEPEND="${RDEPEND}
        python? (
            $(python_gen_cond_dep '
                dev-python/cython[${PYTHON_MULTI_USEDEP}]
            ')
        )"

    pkg_setup() {
        use python && python-single-r1_pkg_setup
    }

    src_configure() {
        econf \
            $(use_enable python pyext) \
            $(use_enable root)
    }


A hybrid: build-time + conditional runtime
==========================================
A fairly common pattern is for Python to be required unconditionally
at build time but only conditionally at runtime.  This happens e.g. when
the package is calling some helper scripts at build time, and optionally
installing Python bindings.  In this case, the build time dependency
is expressed unconditionally, and the runtime dependency is made
USE-conditional:

.. code-block:: bash
   :emphasize-lines: 18,19,23,26,32

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6

    PYTHON_COMPAT=( python3_{6,7,8} )
    PYTHON_REQ_USE="threads(+)"

    inherit waf-utils python-single-r1

    DESCRIPTION="Samba talloc library"
    HOMEPAGE="https://talloc.samba.org/"
    SRC_URI="https://www.samba.org/ftp/${PN}/${P}.tar.gz"

    LICENSE="GPL-3 LGPL-3+ LGPL-2"
    SLOT="0"
    KEYWORDS="~alpha amd64 arm ~arm64 ~hppa ia64 ~m68k ~mips ppc ppc64 ~riscv ~s390 ~sh ~sparc x86 ~amd64-linux ~x86-linux ~x64-macos ~sparc-solaris ~x64-solaris"
    IUSE="+python"
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    RDEPEND="
        ...
        python? ( ${PYTHON_DEPS} )"
    DEPEND="${RDEPEND}
        ...
        ${PYTHON_DEPS}"

    WAF_BINARY="${S}/buildtools/bin/waf"

    src_configure() {
        local extra_opts=(
            $(usex python '' --disable-python)
        )
        waf-utils_src_configure "${extra_opts[@]}"
    }

Note that eclass-exported ``pkg_setup`` is used unconditionally here.


Multiple USE conditions
=======================
Finally, let's give an example of a package where Python is needed
for two independent conditions.  To make it more complex, one of them
applies to build time (tests) while the other to runtime (bindings).

.. code-block:: bash
   :emphasize-lines: 16,19,20,24,27,31-33,38,39

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7,8} )
    inherit cmake python-single-r1

    DESCRIPTION="Sound design and signal processing system for composition and performance"
    HOMEPAGE="https://csound.github.io/"
    SRC_URI="https://dev.gentoo.org/~fordfrog/distfiles/${P}-distributable.tar.xz"

    LICENSE="LGPL-2.1 doc? ( FDL-1.2+ )"
    SLOT="0"
    KEYWORDS="~amd64 ~x86"
    IUSE="python test"
    RESTRICT="!test? ( test )"
    REQUIRED_USE="
        python? ( ${PYTHON_REQUIRED_USE} )
        test? ( ${PYTHON_REQUIRED_USE} )"

    BDEPEND="
        python? ( dev-lang/swig )
        test? ( ${PYTHON_DEPS} )
    "
    RDEPEND="
        python? ( ${PYTHON_DEPS} )
    "

    pkg_setup() {
        if use python || use test ; then
            python-single-r1_pkg_setup
        fi
    }

    src_configure() {
        local mycmakeargs=(
            -DBUILD_PYTHON_INTERFACE=$(usex python)
            -DBUILD_PYTHON_OPCODES=$(usex python)
        )

        cmake_src_configure
    }

Please note that in general, the condition in ``pkg_setup`` must match
the one in ``REQUIRED_USE``, and that one is a superset of conditions
used in dependencies.


Manual install
==============
Some packages do not include Python files in their build systems,
or do not install all of them.  In this case, the necessary files
can be installed via one of the installation helpers.

.. code-block:: bash
   :emphasize-lines: 23,24

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6

    PYTHON_COMPAT=( python2_7 )
    inherit python-single-r1

    DESCRIPTION="Arabic dictionary based on the DICT protocol"
    HOMEPAGE="https://www.arabeyes.org/Duali"
    SRC_URI="mirror://sourceforge/arabeyes/${P}.tar.bz2"

    LICENSE="BSD"
    SLOT="0"
    KEYWORDS="~alpha amd64 ~hppa ~ia64 ~mips ~ppc ~sparc x86"
    IUSE=""
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    DEPEND="${PYTHON_DEPS}"
    RDEPEND="${DEPEND}"

    src_install() {
        python_domodule pyduali
        python_doscript duali dict2db trans2arabic arabic2trans
    }
