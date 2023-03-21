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


PyPI URLs
=========

Modern and legacy URLs
----------------------
The modern form of PyPI URLs include a hash of the distribution file,
e.g.::

    https://files.pythonhosted.org/packages/20/2e/36e46173a288c1c40853ffdb712c67e0e022df0e1ce50b7b1b50066b74d4/gpep517-13.tar.gz

This makes them inconvenient for use in ebuilds, as the hash would have
to be updated manually on every version bump.  For this reason, Gentoo
settled on using the legacy form of URLs instead, e.g.::

    https://files.pythonhosted.org/packages/source/g/gpep517/gpep517-13.tar.gz

It should be noted that legacy URLs are no longer supported and may stop
working at an arbitrary time.  Using the eclass is going to make
the migration to an alternative provider easier than inline URLs.

The path part of a legacy URL for a source package has the general form
of::

    /packages/source/${project::1}/${project}/${filename}

whereas the path for a binary package (wheel) is::

    /packages/${pytag}/${project::1}/${project}/${filename}

In both cases, ``${project}`` refers to the PyPI project name.
Technically, PyPI accepts any name that matches the PyPI project name
after `package name normalization`_.  However, this behavior is not
guaranteed and using the canonical project name is recommended.

The filenames and ``${pytag}`` are described in the subsequent sections.


Source distribution filenames
-----------------------------
The filename of a source distribution (sdist) has the general form of::

    ${name}-${version}.tar.gz

where ``${name}`` is the project name normalized according to `PEP 625`_
and ``${version}`` is the version following `PEP 440`_.  The project
name normalization transforms all uppercase letters to lowercase,
and replaces all contiguous runs of ``._-`` characters with a single
underscore.

For example, package ``Flask-BabelEx`` version ``1.2.3`` would use
the following filename::

    flask_babelex-1.2.3.tar.gz

However, note that source distributions predating PEP 625 are still
commonly found, and they use non-normalized project names, e.g.::

    Flask-BabelEx-1.2.3.tar.gz

In both instances, the top directory inside the archive has the same
name as the archive filename, minus ``.tar.gz`` suffix.  Historically,
``.zip`` distributions were used as well.


Binary distribution filenames
-----------------------------
The filename of a binary distribution (wheel) has the general form of::

    ${name}-${version}-${pytag}-${abitag}-${platformtag}.whl

Similarly to source distributions, ``${name}`` and ``${version}``
specify the normalized project name and version (normalization follows
the same rules and is specified in `binary distribution format`_
specification).

``${pytag}`` is the tag specifying Python version that the wheel was
built for.  This can be ``py2.py3`` or ``py3`` for pure Python wheels,
or e.g. ``cp39`` for CPython 3.9 wheels.  ``${abitag}`` specifies
the appropriate ABI (``none`` for pure Python wheels),
``${platformtag}`` the platform (``any`` for pure Python wheels).

For example, the modern wheel for the aforementioned package would be
named::

    flask_babelex-1.2.3-py3-none-any.whl

However, some distributions use older normalization rules specified
in `PEP 427`_.  In this case, runs of characters other than alphanumeric
characters and dots (``[^\w\d.]+``) are replaced by a single underscore.
Notably, this means that uppercase letters and dots are left in place.
In this normalization, the example wheel is named::

    Flask_BabelEx-1.2.3-py3-none-any.whl


Packages with matching name and version
=======================================
In the most common case, the upstream package will have exactly the same
name as the Gentoo package, and the version numbers will be entirely
compatible.  In this case, it is sufficient to inherit the eclass,
and it will automatically set a suitable default ``SRC_URI``.
The result will be roughly equivalent to::

    SRC_URI="
        https://files.pythonhosted.org/packages/source/${PN::1}/${PN}/${P}.tar.gz
    "
    S=${WORKDIR}/${P}

with ``${P}`` actually using the normalized project name and the Gentoo
version being translated to its PyPI equivalent.  The version
translation performs the following replacements:

  ============= ===========
  Gentoo suffix PyPI suffix
  ============= ===========
  _alpha        a
  _beta         b
  _rc           rc
  _p            .post
  ============= ===========

If the project in question uses a build system that is not compliant
with `PEP 625`_ and has uppercase letters or dots in its name, you may
need to set ``PYPI_NO_NORMALIZE`` to a non-empty value to disable name
normalization, e.g.::

    PYPI_NO_NORMALIZE=1

    inherit distutils-r1 pypi

Note that ``SRC_URI`` is not combined between eclasses and ebuilds.
Should you need to fetch additional files, you need to explicitly append
to the variable or the default will be overwritten, e.g.::

    inherit distutils-r1 pypi

    SRC_URI+="
        https://github.com/pytest-dev/execnet/commit/c0459b92bc4a42b08281e69b8802d24c5d3415d4.patch
            -> ${P}-pytest-7.2.patch
    "


Package with a different name
=============================
If the project name used on PyPI differs from the Gentoo package name,
the ``PYPI_PN`` variable can be used to use another name.  This is
especially useful for project that use uppercase letters or dots
in their names.

For example, a package using lowercase package name in Gentoo and title
case on PyPI could use::

    PYPI_PN=${PN^}

    inherit distutils-r1 pypi

Note that projects whose distfiles do not conform to PEP 625 will also
need to explicitly disable filename normalization.  For example,
a package using dot in its filename and setuptools would use::

    PYPI_NO_NORMALIZE=1
    PYPI_PN=${PN/-/.}

    inherit distutils-r1 pypi


Customizing the generated URL
=============================
The default value may not be suitable for your package if it uses
a different project name than the Gentoo package name, a version number
that needs to be translated differently or the legacy ``.zip`` sdist
format.  The ``pypi_sdist_url`` function can be used to generate URLs
in that case.  Its usage is::

    pypi_sdist_url [--no-normalize] [<project> [<version> [<suffix>]]]

with package defaulting to ``${PYPI_PN}``, version to translated ``${PV}``
and suffix to ``.tar.gz``.  The generated filename uses `PEP 625`_
normalization, unless ``--no-normalize`` is provided
(``PYPI_NO_NORMALIZE`` does not affect explicit function calls).
For example, the Gentoo ``dev-python/markups`` package uses title-case
``Markups`` project name and legacy filename, and so the ebuild needs
to use::

    inherit distutils-r1 pypi

    SRC_URI="$(pypi_sdist_url --no-normalize "${PN^}")"
    S=${WORKDIR}/${P^}

Should the package start using source distributions with normalized
filenames, then only project name would need to be overriden
and the default ``S`` would be correct (``Markups`` and ``markups``
normalize the same)::

    inherit distutils-r1 pypi

    SRC_URI="$(pypi_sdist_url "${PN^}")"

Note that due to project name normalization, the ebuild would also work
without ``SRC_URI`` override.  However, it is recommended to pass
the canonical project name, as normalization is not guaranteed.


Fetching wheels
===============
In very specific cases, it may be necessary to fetch wheels
(i.e. prebuilt Python packages) instead.  The ``pypi_wheel_url``
function is provided to aid this purpose.  Its usage is::

    pypi_wheel_url [--unpack] [<project> [<version> [<python-tag> [<abi-platform-tag>]]]]

with package defaulting to ``${PYPI_PN}``, version to translated ``${PV}``,
python-tag to ``py3`` and abi-platform-tag to ``none-any``
(i.e. indicating a pure Python package).  For example,
``dev-python/ensurepip-setuptools`` does::

    inherit pypi
    SRC_URI="$(pypi_wheel_url "${PN#ensurepip-}")"

Note that wheels are ZIP archives suffixed ``.whl``, and they are not
unpacked by the package manager automatically.  Should you need them
unpacked, you can pass ``--unpack`` option to include a ``->`` operator
that renames the wheel to use ``.whl.zip`` suffix, causing it to be
unpacked.  Remember to add an explicit dependency on ``app-arch/unzip``
in that case.

The ``pypi_wheel_filename`` function is provided to aid getting
the wheel filename.  It has a matching synopsis::

    pypi_wheel_filename [<project> [<version> [<python-tag> [<abi-platform-tag>]]]]


.. _pypi.eclass(5):
   https://devmanual.gentoo.org/eclass-reference/pypi.eclass/index.html
.. _package name normalization:
   https://packaging.python.org/en/latest/specifications/name-normalization/
.. _PEP 625: https://peps.python.org/pep-0625/
.. _PEP 440: https://peps.python.org/pep-0440/
.. _binary distribution format:
   https://packaging.python.org/en/latest/specifications/binary-distribution-format/
.. _PEP 427: https://peps.python.org/pep-0427/
