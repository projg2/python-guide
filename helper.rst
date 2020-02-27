=======================
Common helper functions
=======================

.. index:: python_doexe
.. index:: python_newexe
.. index:: python_doscript
.. index:: python_newscript
.. index:: python_domodule
.. index:: python_doheaders
.. index:: python_scriptinto
.. index:: python_moduleinto

Install helpers
===============
The install helpers are provided commonly for ``python-single-r1``
and ``python-r1`` eclasses.  Their main purpose is to facilitate
installing Python scripts, modules and extensions whenever the package
lacks a build system or the build system is not suited for installing
them.

The API is consistent with the standard ``do*``, ``new*`` and ``*into``
helpers.  There are four kinds of functions provided:

1. ``python_doexe`` and ``python_newexe`` that install executables
   wrapping them via python-exec,
2. ``python_doscript`` and ``python_newscript`` that install Python
   scripts, updating the shebangs and wrapping them via python-exec,
3. ``python_domodule`` that installs Python modules, or recursively
   installs packages (directories),
4. ``python_doheader`` that installs header files to Python-specific
   include directory.

The install path for executables and scripts (1. and 2.) can be adjusted
by calling ``python_scriptinto``.  Note that this actually affects only
the wrapper symlink install path; the actual scripts will be installed
in the standard python-exec script directories.  This also implies that
no two executables can have the same name, even if final directory is
different.  The default install path is ``/usr/bin``.

The install path for modules and packages (3.) can be adjusted
by calling ``python_moduleinto``.  This function accepts either absolute
path or Python parent module name that causes modules to be installed
in an appropriate subdirectory of the site-packages directory.
The default install path is top-level site-packages (equivalent
to ``python_moduleinto .``).

The install path for headers (4.) cannot be adjusted.

``python_doexe`` is generally used to install executables that reference
Python but are not Python scripts.  This could be e.g. a bash script
that calls Python::

    make_wrapper "${PN}.tmp" "${EPYTHON} $(python_get_sitedir)/${PN}/cropgtk.py"
    python_newexe "${ED%/}/usr/bin/${PN}.tmp" "${PN}"
    rm "${ED%/}/usr/bin/${PN}.tmp" || die

Note that you need to ensure that the executable calls correct Python
interpreter itself.

``python_doscript`` is generally used to install Python scripts
to binary directories::

    python_scriptinto /usr/sbin
    python_newscript pynslcd.py pynslcd

It takes care of updating the shebang for you.

``python_domodule`` is used to install Python modules, extensions,
packages, data files and in general anything that lands in site-packages
directory::

    python_moduleinto ${PN}
    python_domodule images application ${MY_PN}.py \
        AUTHORS CHANGES COPYING DEPENDS TODO __init__.py

It is roughly equivalent to ``dodir -r``, except that it byte-compiles
all Python modules found inside it.

``python_doheader`` is used in the very rare cases when Python packages
install additional header files that are used to compile other
extensions::

    python_doheader src/libImaging/*.h


.. index:: python_fix_shebang

Fixing shebangs on installed scripts
====================================
If upstream build system installs Python scripts, it should also update
their shebangs to match the interpreter used for install.  Otherwise,
the scripts could end up being run via another implementation, one
that possible does not have the necessary dependencies installed.
An example of correct shebang is::

    #!/usr/bin/env python3.8

However, if the build system installs a script with ``python3`` or even
``python`` shebang, it needs to be updated.  The ``python_fix_shebang``
function is provided precisely for that purpose.  It can be used to
update the shebang on an installed file::

    src_install() {
        default
        python_fix_shebang "${D}"/usr/bin/sphinxtrain
    }

It can also be used in working directory to update a script that's used
at build time or before it is installed::

    src_prepare() {
        default
        python_fix_shebang openvpn-vulnkey
    }

Finally, it can also be used on a directory to recursively update
shebangs in all Python scripts found inside it::

    src_install() {
        insinto /usr
        doins -r linux-package/*
        dobin linux-package/bin/kitty
        python_fix_shebang "${ED}"
    }

Normally, ``python_fix_shebang`` errors out when the target interpreter
is not compatible with the original shebang, e.g. when you are trying
to install a script with ``python2`` shebang for Python 3.  ``-f``
(force) switch can be used to override that::

    src_prepare() {
        default
        python_fix_shebang -f "${PN}.py"
    }


.. index:: python_optimize

Byte-compiling Python modules
=============================
Python modules are byte compiled in order to speed up their loading.
Byte-compilation is normally done by the build system when the modules
are installed.  However, sometimes packages fail to compile them
entirely, or byte-compile them only partially.  Nowadays, QA checks
detect and report that:

.. code-block:: text

     * This package installs one or more Python modules that are not byte-compiled.
     * The following files are missing:
     *
     *   /usr/lib/pypy2.7/site-packages/_feedparser_sgmllib.pyc
     *   /usr/lib64/python2.7/site-packages/_feedparser_sgmllib.pyc
     *   /usr/lib64/python2.7/site-packages/_feedparser_sgmllib.pyo
     *
     * Please either fix the upstream build system to byte-compile Python modules
     * correctly, or call python_optimize after installing them.  For more
     * information, see:
     * https://wiki.gentoo.org/wiki/Project:Python/Byte_compiling

The eclass provides a ``python_optimize`` function to byte-compile
modules.  The most common way of using it is to call it after installing
the package to byte-compile all modules installed into site-packages::

    src_install() {
        cmake_src_install
        python_optimize
    }

If Python scripts are installed to a non-standard directory, the path
to them can be passed to the function::

    src_install() {
        cd "${S}"/client || die
        emake DESTDIR="${D}" LIBDIR="usr/lib" install
        python_optimize "${D}/usr/lib/entropy/client"
    }


.. index:: python_get_sitedir
.. index:: python_get_includedir
.. index:: python_get_scriptdir
.. index:: python_get_library_path
.. index:: python_get_CFLAGS
.. index:: python_get_LIBS
.. index:: python_get_PYTHON_CONFIG
.. index:: python_is_python3

Querying the implementation information
=======================================
Most of the time, various build systems manage to detect and query
the Python implementation correctly for necessary build details.
Ocassionally, you need to provide those values or override bad detection
results.  For this purpose, the eclasses provide a series of *getters*.

The following generic getters are provided:

- ``python_get_sitedir`` that outputs the absolute path to the target's
  site-packages directory (where Python modules are installed).

- ``python_get_includedir`` that outputs the absolute path
  to the target-specific header directory.

- ``python_get_scriptdir`` that outputs the absolute path
  to the python-exec script directory for the implementation.

The following getters are provided only for CPython targets:

- ``python_get_library_path`` that outputs the absolute path
  to the ``python`` library.

- ``python_get_CFLAGS`` that outputs the C preprocessor flags
  for linking against the Python library (equivalent to ``pkg-config
  --cflags ...``).

- ``python_get_LIBS`` that outputs the linker flags for linking
  against the Python library (equivalent to ``pkg-config --libs ...``).

- ``python_get_PYTHON_CONFIG`` that outputs the absolute path
  to the ``python-config`` executable.

Additionally, the following boolean helper is provided:

- ``python_is_python3`` that returns true (0) if the current interpreter
  implements a variant of Python 3, false (1) if Python 2.

Note that all paths provided by getters include the offset-prefix
(``${EPREFIX}``) already and they are not suitable to passing
to ``*into`` helpers.  If you need to install something, use `install
helpers`_ instead.

.. code-block:: bash

   src_configure() {
       local mycmakeargs=(
           ...
       )
       use python && mycmakeargs+=(
           -DPYTHON_DEST="$(python_get_sitedir)"
           -DPYTHON_EXECUTABLE="${PYTHON}"
           -DPYTHON_INCLUDE_DIR="$(python_get_includedir)"
           -DPYTHON_LIBRARY="$(python_get_library_path)"
       )

       cmake_src_configure
   }


.. code-block:: bash

   python_test() {
       # prepare embedded executable
       emake \
           CC="$(tc-getCC)" \
           PYINC="$(python_get_CFLAGS)" \
           PYLIB="$(python_get_LIBS)" \
           check
   }
