=====================================
python-any-r1 — build-time dependency
=====================================

.. highlight:: bash

The ``python-any-r1`` eclass is used to enable Python support
in packages needing it purely at build time.


Basic use for unconditional Python
==================================
The defining feature of this eclass is that it defines a ``pkg_setup``
phase.  It normally calls ``python_setup`` function in order to find
a suitable Python interpreter, and set the global environment
appropriately.

This means that a most trivial package using an autotools-compatible
build system that needs Python at build time could look like
the following:

.. code-block:: bash
   :emphasize-lines: 6,7,21

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6..8} )
    inherit python-any-r1

    DESCRIPTION="A repository of data files describing media player capabilities"
    HOMEPAGE="https://cgit.freedesktop.org/media-player-info/"
    SRC_URI="https://www.freedesktop.org/software/${PN}/${P}.tar.gz"

    LICENSE="BSD"
    SLOT="0"
    KEYWORDS="~alpha amd64 ~arm ~arm64 ~hppa ~ia64 ~mips ppc ppc64 ~sh ~sparc x86"
    IUSE=""

    RDEPEND=">=virtual/udev-208"
    DEPEND="${RDEPEND}"
    BDEPEND="
        ${PYTHON_DEPS}
        virtual/pkgconfig
    "

This ebuild demonstrates the absolute minimum working code.  Only
the three highlighted lines are specific to Python eclasses, plus
the implicitly exported ``pkg_setup`` phase.


.. index:: python_gen_any_dep; python-any-r1
.. index:: python_check_deps; python-any-r1

Dependencies
============
When depending on other Python packages, USE dependencies need to be
declared in order to ensure that the dependencies would be built against
the Python implementation used for the package.  When Python
dependencies need to be specified, ``${PYTHON_DEPS}`` gets replaced
by a call to ``python_gen_any_dep`` generator and a matching
``python_check_deps()`` function.

The ``python_gen_any_dep`` function accepts a template where literal
``${PYTHON_USEDEP}`` is substituted with appropriate USE dependency.
It generates an any-of (``||``) dependency that requires all
the packages to use the same Python interpreter, for at least one
of the supported implementations.

The ``python_check_deps()`` function needs to be declared by ebuild
in order to test whether the implementation in question is suitable
for building the package.  In particular, it needs to verify whether
the particular branch of the any-of was satisfied, or whether all
dependencies were installed for the current interpreter.  For that
purpose, the function is called with ``PYTHON_USEDEP`` variable declared
to the USE dependency string for the currently tested implementation.

This is best explained using an example:

.. code-block:: bash
   :emphasize-lines: 19-22,25-28

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6
    PYTHON_COMPAT=( python3_{6,7,8} )

    inherit meson python-any-r1

    DESCRIPTION="A file manager for Cinnamon, forked from Nautilus"
    HOMEPAGE="http://developer.linuxmint.com/projects/cinnamon-projects.html"
    SRC_URI="https://github.com/linuxmint/nemo/archive/${PV}.tar.gz -> ${P}.tar.gz"

    LICENSE="GPL-2+ LGPL-2+ FDL-1.1"
    SLOT="0"
    KEYWORDS="amd64 x86"
    IUSE=""

    DEPEND="
        $(python_gen_any_dep '
            dev-python/polib[${PYTHON_USEDEP}]
            dev-python/pygobject:3[${PYTHON_USEDEP}]
        ')
    "

    python_check_deps() {
        has_version "dev-python/polib[${PYTHON_USEDEP}]" &&
        has_version "dev-python/pygobject:3[${PYTHON_USEDEP}]"
    }

This means that the package will work with Python 3.6, 3.7 or 3.8,
provided that its both dependencies have the same implementation
enabled.  The generated ``||`` dep ensures that this is true for
at least one of them, while ``python_check_deps()`` verifies which
branch was satisfied.

It is important to understand that this works correctly only if
``python_gen_any_dep`` and ``python_check_deps()`` match exactly.
Furthermore, for any USE flag combination ``python_gen_any_dep`` must be
called at most once.  In particular, it is invalid to split the above
example into multiple ``python_gen_any_dep`` calls.


Conditional Python use
======================
In some packages, Python is only necessary with specific USE flag
combinations.  This is particularly common when Python is used for
the test suite.  In that case, the dependencies and ``pkg_setup`` call
need to be wrapped in appropriate USE conditions:

.. code-block:: bash
   :emphasize-lines: 16,17,21-27,35

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7} )
    inherit python-any-r1

    DESCRIPTION="Programmable Completion for bash"
    HOMEPAGE="https://github.com/scop/bash-completion"
    SRC_URI="https://github.com/scop/bash-completion/releases/download/${PV}/${P}.tar.xz"

    LICENSE="GPL-2+"
    SLOT="0"
    KEYWORDS="~alpha amd64 arm ~arm64 ~hppa ia64 ~mips ppc ~ppc64 ~s390 ~sh sparc x86 ~amd64-linux ~x86-linux ~ppc-macos ~x64-macos ~x86-macos ~m68k-mint ~sparc-solaris ~sparc64-solaris"
    IUSE="test"
    RESTRICT="!test? ( test )"

    RDEPEND=">=app-shells/bash-4.3_p30-r1:0"
    DEPEND="
        test? (
            ${RDEPEND}
            $(python_gen_any_dep '
                dev-python/pexpect[${PYTHON_USEDEP}]
                dev-python/pytest[${PYTHON_USEDEP}]
            ')
        )"

    python_check_deps() {
        has_version -d "dev-python/pexpect[${PYTHON_USEDEP}]" &&
        has_version -d "dev-python/pytest[${PYTHON_USEDEP}]"
    }

    pkg_setup() {
        use test && python-any-r1_pkg_setup
    }


Additional conditional dependencies
===================================
Another possible case is that Python is required unconditionally
but some dependencies are required only conditionally to USE flags.
The simplest way to achieve that is to use ``${PYTHON_DEPS}`` globally
and ``python_gen_any_dep`` in USE-conditional block, then express
a similar condition in ``python_check_deps()``:

.. code-block:: bash
   :emphasize-lines: 16,19-24,27-30

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=7

    PYTHON_COMPAT=( python3_{6,7} )
    inherit python-any-r1 cmake

    DESCRIPTION="Qt bindings for the Telepathy D-Bus protocol"
    HOMEPAGE="https://telepathy.freedesktop.org/"
    SRC_URI="https://telepathy.freedesktop.org/releases/${PN}/${P}.tar.gz"

    LICENSE="LGPL-2.1"
    SLOT="0"
    KEYWORDS="amd64 ~arm arm64 x86"
    IUSE="test"
    RESTRICT="!test? ( test )"

    BDEPEND="${PYTHON_DEPS}
        test? (
            $(python_gen_any_dep '
                dev-python/dbus-python[${PYTHON_USEDEP}]
            ')
        )
    "

    python_check_deps() {
        use test || return 0
        has_version -b "dev-python/dbus-python[${PYTHON_USEDEP}]"
    }


Multiple sets of conditional dependencies
=========================================
The hardest case for this eclass is to declare multiple Python
dependencies conditional to different USE flags.  While there are
multiple possible ways of doing that, the least error-prone is to move
USE conditional blocks inside ``python_gen_any_dep``:

.. code-block:: bash
   :emphasize-lines: 15,21-27,30-36,39

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6
    PYTHON_COMPAT=( python3_6 )

    inherit gnome2 python-any-r1

    DESCRIPTION="GObject library for accessing the freedesktop.org Secret Service API"
    HOMEPAGE="https://wiki.gnome.org/Projects/Libsecret"

    LICENSE="LGPL-2.1+ Apache-2.0" # Apache-2.0 license is used for tests only
    SLOT="0"
    KEYWORDS="~alpha amd64 arm arm64 ia64 ~mips ppc ppc64 sparc x86"
    IUSE="+introspection test"
    RESTRICT="!test? ( test )"
    # Tests fail with USE=-introspection, https://bugs.gentoo.org/655482
    REQUIRED_USE="test? ( introspection )"

    DEPEND="
        test? (
            $(python_gen_any_dep '
                dev-python/mock[${PYTHON_USEDEP}]
                dev-python/dbus-python[${PYTHON_USEDEP}]
                introspection? ( dev-python/pygobject:3[${PYTHON_USEDEP}] )
            ')
        )
    "

    python_check_deps() {
        if use introspection; then
            has_version --host-root "dev-python/pygobject:3[${PYTHON_USEDEP}]" || return 1
        fi
        has_version --host-root "dev-python/mock[${PYTHON_USEDEP}]" &&
        has_version --host-root "dev-python/dbus-python[${PYTHON_USEDEP}]"
    }

    pkg_setup() {
        use test && python-any-r1_pkg_setup
    }
