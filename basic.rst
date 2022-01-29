=============
Common basics
=============

.. highlight:: bash

The various eclasses in python-r1 try to follow a single design.  You
will probably use more than one of them, so it is worthwhile to shortly
explain the common bits used by all of them, as well as the non-obvious
differences between them.


.. index:: PYTHON_COMPAT

PYTHON_COMPAT
=============
The ``PYTHON_COMPAT`` variable is used by all Python eclasses, and must
be declared in all ebuilds before they are inherited.  It specifies
the list of Python implementations supported by the package.

The valid values are:

- ``pythonX_Y`` for CPython X.Y
- ``pypy3`` for PyPy3.

Typical use::

    PYTHON_COMPAT=( python3_{6,7,8} pypy3 )
    inherit python-single-r1


.. index:: PYTHON_DEPS
.. index:: PYTHON_REQUIRED_USE

PYTHON_DEPS and PYTHON_REQUIRED_USE
===================================
The ``python-any-r1``, ``python-single-r1`` and ``python-r1`` eclasses
all assume that the developer is responsible for explicitly putting
the dependency strings and USE requirements in correct variables.
This is meant to consistently cover packages that use Python
unconditionally and conditionally, at build time and at runtime.

For ``python-single-r1`` and ``python-r1``, the most basic form to use
Python unconditionally is to define the following::

    REQUIRED_USE=${PYTHON_REQUIRED_USE}

    RDEPEND=${PYTHON_DEPS}
    BDEPEND=${RDEPEND}

For ``python-any-r1``, only build-time dependencies are used::

    BDEPEND=${PYTHON_DEPS}

This does not apply to ``distutils-r1`` as it does the above assignment
by default.


.. index:: EPYTHON
.. index:: PYTHON

Python environment
==================
The eclasses commonly use the concept of *Python environment*.  This
means a state of environment enforcing a particular Python
implementation.  Whenever the ebuild code is run inside this
environment, two variables referring to the specific Python interpreter
are being exported:

- ``EPYTHON`` containing the interpreter's basename (also used
  as the implementation identifier), e.g. ``python3.10``
- ``PYTHON`` containing the absolute final path to the interpreter,
  e.g. ``/usr/bin/python3.10``

The full path should only be used to provide the value that should
be embedded in the installed programs, e.g. in the shebangs.
For spawning Python during the build, ``EPYTHON`` is preferable.

.. Warning::

   Using full path rather than the basename will bypass the virtualenv
   created by ``distutils-r1.eclass`` in PEP 517 mode.  This may cause
   failures to import Python modules, or use of the previous installed
   version rather than the just-built one.  Using ``${EPYTHON}``
   resolves these problems.

Wrappers for ``python``, ``pythonN`` and some common tools are provided
in PATH, and ``/usr/bin/python`` etc. also enforce the specific
implementation via python-exec (for programs that hardcode full path).

The environment can be either established in local scope, or globally.
The local scope generally applies to multi-impl packages, and is created
either by calls to ``python_foreach_impl`` from ``python-r1``, or inside
sub-phase functions in ``distutils-r1``.  The global scope setup is done
via calling ``python_setup``, either directly or via default
``pkg_setup`` in ``python-any-r1`` and ``python-single-r1``.
