======================
QA checks and warnings
======================

.. highlight:: text

This section explains Python-related QA checks and the resulting QA
warnings that can be output while running the package manager or related
tooling.


Improved QA warning reporting in Portage
========================================
Normally, Portage outputs QA warnings at specific phases of the build
process.  They are usually interspersed with other verbose output,
and they are easy to miss, especially when building multiple packages
in a single batch.

To make them easier to catch, Portage's elog system can be used
to repeat all the QA warnings once emerge exits.  The required "echo"
module is already enabled by default, however it skips QA warnings
by default.  To change that, set in your ``make.conf``:

.. code-block:: bash

    PORTAGE_ELOG_CLASSES="log warn error qa"

For more information on using Portage's elog system, please refer
to make.conf.example_ included in the Portage distribution.


Compiled bytecode-related warnings
==================================
To improve performance, the Python interpreter compiles Python sources
into bytecode.  CPython and PyPy3 feature three optimization levels
that impact the bytecode size:

1. no optimizations (the default)

2. ``-O`` that removes assert statements from code

3. ``-OO`` that removes assert statements and docstrings

Normally, the compiled bytecode is stored on disk in ``.pyc`` files
(historically, Python 2 has also used ``.pyo`` files).
When these files are present and up-to-date, Python loads
the precompiled bytecode from them rather than creating it from
the source code, improving module loading time.  When they are missing,
Python normally creates them automatically if it has write permissions
to the respective directory.

In Gentoo, we aim for packages to always byte-compile all their Python
files to all supported optimization levels.  Besides improving module
loading performance, it ensures that the Python interpreters will not
attempt to write compiled bytecode themselves, effectively creating
files that are not monitored by the package manager (and therefore e.g.
are not removed when the respective package is uninstalled) or causing
sandbox violations while building other packages.

The Gentoo repository features a QA check to ensure that all installed
Python modules are byte-compiled for all optimization levels supported
by the respective Python interpreter, and that no stray compiled files
exist.  This check is implemented using an auxiliary command found
in ``app-portage/gpep517-7`` and newer.  This check can also be run
manually (with machine-readable output) using e.g.::

    $ python3.10 -m gpep517 verify-pyc --destdir=/tmp/portage/dev-python/trimesh-3.12.9/image
    missing:/usr/lib/python3.10/site-packages/trimesh/resources/templates/__pycache__/blender_boolean.cpython-310.opt-1.pyc:/usr/lib/python3.10/site-packages/trimesh/resources/templates/blender_boolean.py
    missing:/usr/lib/python3.10/site-packages/trimesh/resources/templates/__pycache__/blender_boolean.cpython-310.opt-2.pyc:/usr/lib/python3.10/site-packages/trimesh/resources/templates/blender_boolean.py
    missing:/usr/lib/python3.10/site-packages/trimesh/resources/templates/__pycache__/blender_boolean.cpython-310.pyc:/usr/lib/python3.10/site-packages/trimesh/resources/templates/blender_boolean.py


Modules are not byte-compiled
-----------------------------
The most common QA warning that can be noticed while building packages
indicates that at least some of the expected ``.pyc`` files are missing.
For example::

     * QA Notice: This package installs one or more Python modules that are
     * not byte-compiled.
     * The following files are missing:
     *
     *   /usr/lib/python3.10/site-packages/trimesh/resources/templates/__pycache__/blender_boolean.cpython-310.opt-1.pyc
     *   /usr/lib/python3.10/site-packages/trimesh/resources/templates/__pycache__/blender_boolean.cpython-310.opt-2.pyc
     *   /usr/lib/python3.10/site-packages/trimesh/resources/templates/__pycache__/blender_boolean.cpython-310.pyc

     * QA Notice: This package installs one or more Python modules that are
     * not byte-compiled.
     * The following files are missing:
     *
     *   /usr/lib/python3.10/site-packages/blueman/__pycache__/Constants.cpython-310.opt-2.pyc
     *   /usr/lib/python3.10/site-packages/blueman/__pycache__/DeviceClass.cpython-310.opt-2.pyc
     *   /usr/lib/python3.10/site-packages/blueman/__pycache__/Functions.cpython-310.opt-2.pyc
     *   /usr/lib/python3.10/site-packages/blueman/__pycache__/Sdp.cpython-310.opt-2.pyc
     [...]

There are three common causes for these warnings:

1. The package's build system nor the ebuild do not byte-compile
   installed Python modules — the warning lists all optimization levels
   for all installed modules.

2. The package's build system byte-compiles installed modules only for
   a subset of optimization levels — the warning lists all modules
   but only for a subset of levels (the second example in the example
   snippet).

3. The package installs ``.py`` files with incorrect syntax that can not
   be byte-compiled and usually trigger syntax errors during the install
   phase (the first example in the above snippet).

For the first two cases, the better solution is to patch the respective
build system to perform byte-compilation for all optimization levels.
An acceptable workaround is to call ``python_optimize`` from the ebuild
(note that in PEP517 mode, distutils-r1 does that unconditionally).

For the third case, the only real solution is to submit a fix upstream
that renames files that do not contain valid Python modules to use
another suffix.  For example, the template triggering the QA warning
in trimesh package could be renamed from ``.py`` to ``.py.tmpl``.


Stray compiled bytecode
-----------------------
The following QA warning indicates that there are stray ``.pyc`` files
that are not clearly matching any installed Python module-implementation
pair::

     * QA Notice: This package installs one or more compiled Python modules
     * that do not match installed modules (or their implementation).
     * The following files are stray:
     *
     *   /usr/lib/python3.10/site-packages/SCons/Tool/docbook/__pycache__/__init__.cpython-35.pyc
     *   /usr/lib/python3.10/site-packages/SCons/Tool/docbook/__pycache__/__init__.cpython-36.pyc
     *   /usr/lib/python3.10/site-packages/SCons/Tool/docbook/__pycache__/__init__.cpython-38.pyc

There are two common causes for this:

1. The package is shipping precompiled ``.pyc`` files and installing
   them along with ``.py`` modules.  The ebuild should remove the stray
   files in ``src_prepare`` then.

2. The ebuild is attempting to remove some ``.py`` files after they have
   been byte-compiled.  It needs to be modified to either remove them
   prior to the byte-compilation stage, or to fix the build system
   not to install them in the first place.


.. _make.conf.example:
   https://gitweb.gentoo.org/proj/portage.git/tree/cnf/make.conf.example#n330
