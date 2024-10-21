================
Migration guides
================


.. index:: PYTHON_MULTI_USEDEP

Migrating from old PYTHON_USEDEP syntax in python-single-r1
===========================================================
Prior to February 2020, ``python-single-r1`` used to provide a single
``PYTHON_USEDEP`` variable alike the two other eclasses.  However,
getting it to work correctly both on single-impl and multi-impl packages
required a gross hack.

The current eclass API requires using ``python_gen_cond_dep`` function
to generate multi-impl deps instead, with ``PYTHON_USEDEP`` serving
as a placeholder.  Single-impl deps can be expressed with
``PYTHON_SINGLE_USEDEP`` that can be used either as placeholder,
or directly as a variable.

During a transitional period, ``PYTHON_USEDEP`` was banned entirely
and ``PYTHON_MULTI_USEDEP`` was used instead.  As of EAPI 8,
the opposite is true — ``PYTHON_USEDEP`` is to be used,
and ``PYTHON_MULTI_USEDEP`` was removed.

The recommended rule of thumb for migrating old ebuilds is to:

1. Replace all instances of ``${PYTHON_USEDEP}`` for simple-impl deps
   with ``${PYTHON_SINGLE_USEDEP}``.  If you don't know the type
   of given dep, dependency check (repoman, pkgcheck) will tell you
   if you chose wrong.

2. Wrap the dependencies using ``${PYTHON_USEDEP}`` in a single
   ``python_gen_cond_dep`` block (reordering may be desirable).

3. Run ``pkgcheck scan`` or ``repoman full``.  If you get syntax errors,
   you probably missed ``python_gen_cond_dep`` or did not escape
   the ``$`` in placeholder properly.  If you get unmatched dependency,
   you probably got single-impl vs. multi-impl wrong.

This way, you can generally convert ebuilds using trial-and-error
method.


.. index:: EAPI 8

Migrating from EAPI 7 to EAPI 8
===============================
EAPI 8 has banned everything that's been deprecated in EAPI 7, as well
as some other obsolete stuff.  The following table lists all banned
things along with their suggested replacements.

  +-------------------------------+------------------------------------+
  | Deprecated thing              | Replacement                        |
  +===============================+====================================+
  | Private eclass API                                                 |
  +-------------------------------+------------------------------------+
  | python_export                 | python_setup / getters             |
  +-------------------------------+------------------------------------+
  | python_wrapper_setup          | python_setup                       |
  +-------------------------------+------------------------------------+
  | Obsolete API                                                       |
  +-------------------------------+------------------------------------+
  | distutils_install_for_testing | no argument (``--via-root``)       |
  | ``--via-home``                | or ``--via-venv``                  |
  +-------------------------------+------------------------------------+
  | python_gen_usedep             | python_gen_cond_dep                |
  +-------------------------------+------------------------------------+
  | PYTHON_MULTI_USEDEP           | PYTHON_USEDEP                      |
  +-------------------------------+------------------------------------+
  | mydistutilsargs rename                                             |
  +-------------------------------+------------------------------------+
  | mydistutilsargs               | DISTUTILS_ARGS                     |
  +-------------------------------+------------------------------------+
  | Post-Python 2 cleanup                                              |
  +-------------------------------+------------------------------------+
  | python_gen* -2 / python2      | remove entirely                    |
  | / pypy                        |                                    |
  +-------------------------------+------------------------------------+
  | python_gen* -3                | make unconditional                 |
  +-------------------------------+------------------------------------+
  | python_is_python3             | always assume true                 |
  +-------------------------------+------------------------------------+

The changes can be split roughly into four groups: ban of now-private
eclass API, ban of obsolete API functions, mydistutilsargs rename
and bans related to post-Python 2 cleanup.

The private eclass API part involves ``python_export``
and ``python_wrapper_setup``.  Both were deprecated in March 2020,
and they were never covered in this guide.  The former was historically
used to get information about the Python interpreter (either the current
``${EPYTHON}`` or an arbitrary choice), the latter to create the wrapper
directory containing ``python`` and other executables.

When the functions were used to establish a Python build environment,
the replacement for both is a single ``python_setup`` call.  When
``python_export`` was used to grab additional details about the Python
interpreter, the various ``python_get*`` functions should be used
instead.

.. code-block:: bash

    src_configure() {
        # ...

        # OLD:
        local PYTHON_INCLUDEDIR PYTHON_LIBPATH
        python_export PYTHON_INCLUDEDIR PYTHON_LIBPATH
        mycmakeargs+=(
            -DPython3_INCLUDE_DIR="${PYTHON_INCLUDEDIR}"
            -DPython3_LIBRARY="${PYTHON_LIBPATH}"
        )

        # NEW:
        mycmakeargs+=(
            -DPython3_INCLUDE_DIR="$(python_get_includedir)"
            -DPython3_LIBRARY="$(python_get_library_path)"
        )
    }

The second group involves sundry API that were deprecated earlier.
These are:

1. ``distutils_install_for_testing --via-home`` layout that stopped
   working correctly at some point.  The default ``--via-root`` should
   work most of the time, and ``-via-venv`` replace the remaining cases
   for the removed layout.

2. ``python_gen_usedep`` function that was historically used to generate
   partial USE dependencies, and was generally combined with
   ``REQUIRED_USE`` to force specific (usually old) Python interpreters
   for specific features.  This was really ugly.  Nowadays, you should
   really use ``python_gen_cond_dep`` instead.

3. ``PYTHON_MULTI_USEDEP`` placeholder that was temporarily used
   in python-single-r1 ebuilds.  ``PYTHON_USEDEP`` is equivalent now.

The third group is a sole rename of ``mydistutilsargs`` variable.
Since you usually need to pass the same arguments in all phase
functions, this variable was not really used in local scope.  It has
been renamed to uppercase ``DISTUTILS_ARGS`` to follow the common
pattern for global scope variables.

Finally, the fourth group involves banning some of the features that
were specifically used in order to support distinguish between Python 2
and Python 3.  This is meant to force cleaning up old cruft from
ebuilds.  It comes in three parts:

1. Banning arguments to ``python_gen*`` that reference Python 2
   (e.g. ``-2``, ``python2*``, ``python2_7``, ``pypy``).  Since Python 2
   is no longer supported in the relevant code paths, the relevant calls
   should just be removed.

2. Banning the ``-3`` short-hand to ``python_gen*``.  Since all
   supported interpreters are compatible with Python 3 now, the relevant
   code should be made unconditional.  Note that ``python3*`` is still
   useful, as it distinguishes CPython from PyPy3.

3. Banning the ``python_is_python3`` function.  Since the removal
   of Python 2 support, it always evaluated to true.

All the aforementioned replacements are available in all EAPIs.


Migrating to PEP 517 builds
===========================
As of January 2022, the ``distutils-r1`` can use PEP 517 build backends
instead of calling setuptools directly.  The new mode is particularly
useful for:

- packages using flit and poetry, as a better replacement for
  the deprecated ``dev-python/pyproject2setuppy`` hack

- packages using other PEP 517 build systems (such as pdm) that are not
  supported in legacy mode at all

- packages using setuptools without ``setup.py``

- packages using plain distutils, as the mode handles the switch from
  deprecated stdlib distutils to the version vendored in setuptools
  safely

The PEP 517 mode provides the test phase with venv-style installed
package tree (alike ``distutils_install_for_testing --via-venv``)
that should make testing more streamlined.

Unfortunately, the new mode can cause issues with customized distutils
and setuptools build systems.  It is important to verify the installed
file list after the migration.  Packages that require custom configure
phases or passing arguments are not supported at the moment.

For simple packages, the migration consists of:

1. Adding ``DISTUTILS_USE_PEP517`` above the inherit line.  The value
   indicates the build system used, e.g. ``flit``, ``poetry``,
   ``setuptools`` (used also for distutils).

2. Removing ``DISTUTILS_USE_SETUPTOOLS``.  If the previous value was
   ``rdepend`` (and indeed a runtime dependency is required), then
   ``dev-python/setuptools`` needs to be explicitly added to
   ``RDEPEND``.

3. Removing ``distutils_install_for_testing`` and/or ``--install``
   option to ``distutils_enable_tests``.  This should no longer be
   necessary and tests should work out of the box.


.. index:: multipart
.. index:: python-multipart

multipart vs. python-multipart packages
=======================================
Prior to November 2024, two PyPI packages claimed the ``multipart``
import name: multipart_ and python-multipart_.  Originally, Gentoo
packaged only the latter, as it was necessary to satisfy dependencies.
However, with the former being listed as the recommended replacement
for the `deprecated cgi module`_, new packages started depending on it.

At this point, Gentoo decided to package multipart as well,
and to rename python-multipart in anticipation of upstream rename
already in progress, in ``>=dev-python/python-multipart-0.0.12-r100``.
However, it should be noted that the Gentoo rename does not install
a compatibility loader to permit importing it via ``multipart`` import
name, as that would work only if ``dev-python/multipart``
was not installed.  Instead, packages need to be explicitly patched
to use the new import name of ``python_multipart``.

Therefore, the dependencies on these packages should be handled
in the following manner:

1. If the package depends on multipart, a dependency
   on ``dev-python/multipart`` should be added.

2. If the package bundles multipart, the dependency should be
   unbundled.

3. If the package depends on python-multipart and uses the new import
   name (i.e. ``import python_multipart``), a dependency
   on ``>=dev-python/python-multipart-0.0.12-r100`` (or a newer version)
   should be added.

4. If the package depends on python-multipart and uses the old import
   name (i.e. ``import multipart``), it should be patched to use
   the new import name instead, and the dependency should be added
   as above.

The following snippet provides an example of patching
the python-multipart import:

.. code-block:: bash

    RDEPEND="
        >=dev-python/python-multipart-0.0.12-r100[${PYTHON_USEDEP}]
    "

    src_prepare() {
        distutils-r1_src_prepare

        # python-multipart package renamed in Gentoo to python_multipart
        find -name "*.py" -exec sed \
            -e "s:from multipart:from python_multipart:" \
            -e "s:import multipart:import python_multipart as multipart:" \
            -i {} + || die
    }


.. _multipart: https://pypi.org/project/multipart/
.. _python-multipart: https://pypi.org/project/python-multipart/
.. _deprecated cgi module: https://docs.python.org/3.12/library/cgi.html
