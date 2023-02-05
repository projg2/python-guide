======================================
pypi — helper eclass for PyPI archives
======================================

.. highlight:: bash

The ``pypi`` eclass is a small eclass to facilitate fetching sources
from PyPI.  It abstracts away the complexity of PyPI URLs, and makes it
easier to adapt ``SRC_URI`` to their future changes.

Please note that PyPI archives are not always the best choice
for distfiles.  In particular, they frequently are missing tests
and other files important to Gentoo packaging.  Should that be the case,
other archives should be used.  Read the :ref:`source archives` section
for more information.

Eclass reference: `pypi.eclass(5)`_


Packages with matching name and version
=======================================
In the most common case, the upstream package will have exactly the same
name as the Gentoo package, and the version numbers will be entirely
compatible.  In this case, it is sufficient to inherit the eclass,
and it will automatically set a suitable default ``SRC_URI``.  The URI
will be equivalent to::

    SRC_URI="
        https://files.pythonhosted.org/packages/source/${PN::1}/${PN}/${P}.tar.gz
    "

Note that ``SRC_URI`` is not combined between eclasses and ebuilds.
Should you need to fetch additional files, you need to explicitly append
to the variable or the default will be overwritten, e.g.::

    inherit distutils-r1 pypi

    SRC_URI+="
        https://github.com/pytest-dev/execnet/commit/c0459b92bc4a42b08281e69b8802d24c5d3415d4.patch
            -> ${P}-pytest-7.2.patch
    "


Customizing the generated URL
=============================
The default value may not be suitable for your package if it uses
a different project name, version numbers that are incompatible with
Gentoo or the legacy ``.zip`` sdist format.  The ``pypi_sdist_url``
function can be used to generate URLs in that case.  Its usage is::

    pypi_sdist_url [<project> [<version> [<suffix>]]]

with package defaulting to ``${PN}``, version to ``${PV}`` and suffix
to ``.tar.gz``.  For example, the Gentoo ``dev-python/markups`` package
uses title-case ``Markups`` project name, and so the ebuild uses::

    inherit distutils-r1 pypi

    SRC_URI="$(pypi_sdist_url "${PN^}")"


Fetching wheels
===============
In very specific cases, it may be necessary to fetch wheels
(i.e. prebuilt Python packages) instead.  The ``pypi_wheel_url``
function is provided to aid this purpose.  Its usage is::

    pypi_wheel_url [<project> [<version> [<python-tag> [<abi-platform-tag>]]]]

with package defaulting to ``${PN}``, version to ``${PV}``, python-tag
to ``py3`` and abi-platform-tag to ``none-any`` (i.e. indicating a pure
Python package).  For example, ``dev-python/ensurepip-setuptools``
does::

    inherit pypi
    SRC_URI="$(pypi_wheel_url "${PN#ensurepip-}")"

Note that wheels are ZIP archives suffixed ``.whl``, and they are not
unpacked by the package manager automatically.  You either need to
unzip it explicitly or use ``->`` to rename it, e.g. by appending
``.zip`` suffix.  Remember to add an explicit dependency
on ``app-arch/unzip`` as well.

The ``pypi_wheel_filename`` function is provided to aid getting
the wheel filename.  It has a matching synopsis::

    pypi_wheel_filename [<project> [<version> [<python-tag> [<abi-platform-tag>]]]]


.. _pypi.eclass(5):
   https://devmanual.gentoo.org/eclass-reference/pypi.eclass/index.html
