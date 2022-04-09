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

1. `PEP 420`_ namespaces implemented in Python 3.3 and newer,

2. Using pkgutil_ standard library module,

3. Using `namespace package support in setuptools`_ (discouraged).

PEP 420 namespaces are created implicitly when a package directory
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

More general information on the topic can be found under `packaging
namespace packages`_ in Python Packaging User Guide.


Determining whether namespaces are used
---------------------------------------
The exact method of detecting namespace packages depends on the type
of namespace used.

PEP 420 namespaces can generally be recognized by the lack
of ``__init__.py`` in an installed package directory.  However, since
they do not require any specific action, distinguishing them is not very
important.

pkgutil namespaces can be recognized through the content of their
``__init__.py``.  Generally, you should find it suspicious if is
the only file in a top-level package directory, and if the name of this
directory is less specific than the package name (e.g. ``zope`` for
``zope.interface``, ``ruamel`` for ``ruamel.yaml``).  If you miss this,
then you will learn about the namespace from package collisions
on the respective ``__init__.py``.

setuptools namespaces usually do not install ``__init__.py`` but
do install a ``.pth`` file instead.  The distutils-r1 eclass detects
this automatically and prints a warning.  Prior to installation,
they can also be recognized by ``namespace_packages`` option
in ``setup.py`` or ``setup.cfg``.


Adding new namespace packages to Gentoo
---------------------------------------
If the package uses PEP 420 namespaces, no special action is required.
Per PEP 420 layout, the package must not install ``__init__.py`` files
for namespaces.

If the package uses one of the other layouts, their respective files
must be removed from the install tree.

For pkgutil namespace, its ``__init__.py`` should be removed after
the PEP 517 build phase:

.. code-block:: bash

    python_compile() {
        distutils-r1_python_compile
        rm "${BUILD_DIR}/install$(python_get_sitedir)"/jaraco/__init__.py || die
    }

The equivalent code for the legacy eclass mode is:

.. code-block:: bash

    python_install() {
        rm "${BUILD_DIR}"/lib/jaraco/__init__.py || die
        distutils-r1_python_install
    }

For setuptools namespace, the ``.pth`` file should be removed instead:

.. code-block:: bash

    python_compile() {
        distutils-r1_python_compile
        find "${BUILD_DIR}" -name '*.pth' -delete || die
    }

The setuptools code for the legacy mode is:

.. code-block:: bash

    python_install_all() {
        distutils-r1_python_install_all
        find "${D}" -name '*.pth' -delete || die
    }


Legacy namespace packages in Gentoo
-----------------------------------
Historically, Gentoo has used ``dev-python/namespace-*`` packages
to support namespaces.  This method is deprecated and it is in process
of being retired.


.. _PEP 420: https://www.python.org/dev/peps/pep-0420/

.. _pkgutil: https://docs.python.org/3/library/pkgutil.html

.. _namespace package support in setuptools:
   https://setuptools.readthedocs.io/en/latest/setuptools.html#namespace-packages

.. _packaging namespace packages:
   https://packaging.python.org/guides/packaging-namespace-packages/
