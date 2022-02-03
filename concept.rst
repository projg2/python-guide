=================
Advanced concepts
=================

.. highlight:: python

Namespace packages
==================

Hierarchical package structure
------------------------------
Traditionally, Python packages were organized into a hierarchical
structure with modules and subpackages being located inside the parent
package directory.  When submodules are imported, they are represented
as attributes on the parent module.  Consider the following session::

    >>> import sphinx.addnodes
    >>> sphinx
    <module 'sphinx' from '/usr/lib/python3.8/site-packages/sphinx/__init__.py'>
    >>> sphinx.addnodes
    <module 'sphinx.addnodes' from '/usr/lib/python3.8/site-packages/sphinx/addnodes.py'>

This works fine most of the time.  However, it start being problematic
when multiple Gentoo packages install parts of the same top-level
package.  This may happen e.g. with some plugin layouts where plugins
are installed inside the package.  More commonly, it happens when
upstream wishes all their packages to start with a common component.

This is the case with Zope framework.  Different Zope packages share
common ``zope`` top-level package.  ``dev-python/zope-interface``
installs into ``zope.interface``, ``dev-python/zope-event``
into ``zope.event``.  For this to work using the hierarchical layout,
a common package has to install ``zope/__init__.py``, then other Zope
packages have to depend on it and install sub-packages inside that
directory.  As far as installed packages are concerned, this is entirely
doable.

The real problem happens when we wish to test a freshly built package
that depends on an installed package.  In that case, Python imports
``zope`` from build directory that contains only ``zope.interface``.
It will not be able to import ``zope.event`` that is installed in system
package directory::

    >>> import zope.interface
    >>> zope
    <module 'zope' from '/tmp/portage/dev-python/zope-interface-4.7.1/work/zope.interface-4.7.1-python3_8/lib/zope/__init__.py'>
    >>> zope.interface
    <module 'zope.interface' from '/tmp/portage/dev-python/zope-interface-4.7.1/work/zope.interface-4.7.1-python3_8/lib/zope/interface/__init__.py'>
    >>> import zope.event
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    ModuleNotFoundError: No module named 'zope.event'

Now, this could be worked around by copying all other subpackages back
to the build directory.  However, there is a better solution.


Namespace package structure
---------------------------
Unlike traditional packages, namespace packages act as a kind of proxy.
They are not strictly bound to the containing directory, and instead
permit loading subpackages from all directories found in module search
path.  If we make ``zope`` a namespace package, we can import both
the locally built ``zope.interface`` and system ``zope.event``
packages::

    >>> import zope.interface
    >>> import zope.event
    >>> zope
    <module 'zope' (namespace)>
    >>> zope.interface
    <module 'zope.interface' from '/tmp/portage/dev-python/zope-interface-4.7.1/work/zope.interface-4.7.1-python3_8/lib/zope/interface/__init__.py'>
    >>> zope.event
    <module 'zope.event' from '/usr/lib/python3.8/site-packages/zope/event/__init__.py'>

There are three common methods of creating namespace packages:

1. PEP-0420_ namespaces implemented in Python 3.3 and newer,

2. Using pkgutil_ standard library module,

3. Using `namespace package support in setuptools`_ (discouraged).

PEP-0420 namespaces are created implicitly when a package directory
does not contain ``__init__.py`` file.  While earlier versions
of Python (including Python 2.7) ignored such directories and did not
permit importing Python modules within them, Python 3.3 imports such
directories as namespace packages.

pkgutil namespaces use ``__init__.py`` with the following content::

    __path__ = __import__('pkgutil').extend_path(__path__, __name__)

setuptools namespace can use ``__init__.py`` with the following
content::

    __import__('pkg_resources').declare_namespace(__name__)

Alternatively, setuptools normally installs a ``.pth`` file that is
automatically loaded by Python and implicitly injects the namespace
into Python.

Both pkgutil and setuptools namespaces are portable to all versions
of Python.

PEP-0420 and pkgutil namespaces are considered mutually compatible,
while setuptools namespaces are considered incompatible with them.
It is recommended not to mix different methods within a single
namespace.

More general information on the topic can be found under `packaging
namespace packages`_ in Python Packaging User Guide.


Namespace packages in Python 3
------------------------------
Since all supported Python versions in Gentoo support PEP-0420
namespaces, the other two methods are technically unnecessary.  However,
the incompatibility between pkg_resources namespaces and the other two
methods makes removing them non-trivial.

If all packages within the namespace are using only pkgutil-style
namespaces, you can safely remove the dependencies on the package
providing the namespace and the package itself.  Even partial removal
should not cause any issues.  However, if the package was explicitly
provided upstream, note that some packages may carry an explicit
dependency on it and that dependency would need to be removed or made
conditional to Python < 3.3.  You will also need to strip colliding
``__init__.py`` files.

If setuptools-style namespace are used, the namespace packages need
to remain as-is for the time being, as otherwise tests relying
on the namespaced packages are going to be broken.  We have not yet
conceived a way forward for them.


Packaging pkgutil-style namespaces in Gentoo
--------------------------------------------
Normally all packages using the same pkgutil-style namespace install
its ``__init__.py`` file causing package collisions.  As having this
file is no longer necessary for Python 3.3 and newer, the recommended
solution is to strip it before installing the package.  The presence
of this file is harmless during build and testing.

.. code-block:: bash

    python_install() {
        rm "${BUILD_DIR}"/lib/jaraco/__init__.py || die
        distutils-r1_python_install
    }


Packaging setuptools-style namespaces in Gentoo
-----------------------------------------------
Similar approach is used for setuptools-style namespace packages.
The only differences are in ``__init__.py`` code and removal method.

The ``dev-python/namespace-<name>`` package for setuptools-style
namespace should use the following code:

.. code-block:: bash
   :emphasize-lines: 24-27,31

    # Copyright 1999-2020 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=6

    PYTHON_COMPAT=( pypy3 python{2_7,3_{6,7,8}} )
    inherit python-r1

    DESCRIPTION="Namespace package declaration for zope"
    HOMEPAGE="https://wiki.gentoo.org/wiki/Project:Python/Namespace_packages"
    SRC_URI=""

    LICENSE="public-domain"
    SLOT="0"
    KEYWORDS="~alpha amd64 arm arm64 hppa ia64 ~m68k ~mips ppc ppc64 s390 ~sh sparc x86 ~amd64-linux ~x86-linux ~ppc-macos ~x64-macos ~x86-macos ~sparc-solaris ~sparc64-solaris ~x64-solaris ~x86-solaris"
    IUSE=""
    REQUIRED_USE="${PYTHON_REQUIRED_USE}"

    RDEPEND="dev-python/setuptools[${PYTHON_USEDEP}]
        ${PYTHON_DEPS}"
    DEPEND="${PYTHON_DEPS}"

    src_unpack() {
        mkdir -p "${S}"/zope || die
        cat > "${S}"/zope/__init__.py <<-EOF || die
            __import__('pkg_resources').declare_namespace(__name__)
        EOF
    }

    src_install() {
        python_foreach_impl python_domodule zope
    }

Setuptools normally do not install ``__init__.py`` files but ``*.pth``
files that do not collide.  It is therefore easy to miss them but they
can cause quite a mayhem.  Therefore, remember to strip them.

In PEP 517 mode, the stripping needs to happen after the compile phase
to ensure that tests work correctly:

.. code-block:: bash

    python_compile() {
        distutils-r1_python_compile
        find "${BUILD_DIR}" -name '*.pth' -delete || die
    }

In legacy mode, it should be done after the install phase:

.. code-block:: bash

    python_install_all() {
        distutils-r1_python_install_all
        find "${D}" -name '*.pth' -delete || die
    }


.. _PEP-0420: https://www.python.org/dev/peps/pep-0420/

.. _pkgutil: https://docs.python.org/3/library/pkgutil.html

.. _namespace package support in setuptools:
   https://setuptools.readthedocs.io/en/latest/setuptools.html#namespace-packages

.. _packaging namespace packages:
   https://packaging.python.org/guides/packaging-namespace-packages/
