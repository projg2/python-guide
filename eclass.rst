================================
Choosing between Python eclasses
================================

Overview
========
The python-r1 eclass suite features 5 eclasses in total:

1. ``python-utils-r1.eclass`` that provides utility functions common
   to all eclasses.  This eclass is rarely inherited directly.

2. ``python-any-r1.eclass`` that is suitable for packages using Python
   purely at build time.

3. ``python-single-r1.eclass`` that provides a base for simpler packages
   installing Python scripts and alike.

4. ``python-r1.eclass`` that provides a base for more complex packages,
   particularly these installing Python modules.

5. ``distutils-r1.eclass`` that provides convenient phase functions
   and helpers for packages that primarily involve installing Python
   files.

.. figure:: diagrams/eclass.svg

    Inheritance graph of python-r1 suite eclasses.

As a rule of thumb, the best eclass to use is the one that makes
the ebuild the simplest while meeting its requirements.  A more detailed
process involves:

1. Determining whether Python is used purely at build time,
   or at runtime as well.  In the former case, ``python-any-r1``
   is the right choice.

2. Determining whether single-impl or multi-impl approach is more
   appropriate.  For the former, ``python-single-r1`` is the correct
   base eclass.  For the latter, ``python-r1``.

3. Determining whether the ebuild benefits from using ``distutils-r1``.
   If it does, this eclass should be use instead (potentially along
   with ``DISTUTILS_SINGLE_IMPL`` to switch the underlying eclass).


Build time vs runtime use
=========================
The first basis for choosing Python eclass is whether Python is used
merely at build time or at runtime as well.

A runtime use occurs if the package explicitly needs Python to be
installed along with it, in order for it to function correctly.  This
generally happens if the package installs Python modules, extensions,
scripts, or executables calling the Python interpreter or linking
to libpython.  This also applies to bash scripts or other executables
that call python inline.

A build time use occurs if the package calls the Python interpreter
or any kind of aforementioned executables during package's build
(or install) phases.

If the package uses Python purely at build time, the ``python-any-r1``
eclass is appropriate.  Otherwise, ``python-single-r1``, ``python-r1``
or ``distutils-r1`` are to be used.

A specific exception to that rule is when the package is only calling
external Python scripts directly (i.e. not via ``python /usr/bin/foo``).
If the called executables can be considered fully contained
dependency-wise, there is no need to use an eclass.

For example, when using ``dev-util/meson`` to build a package, there is
no need to use a Python eclass since Meson abstracts away its Pythonic
implementation details and works as a regular executable for your
packages.  However, ``dev-util/scons`` requires Python eclass since it
loads Python code from the package and a compatible Python version must
be enforced.


Single-impl vs multi-impl
=========================
The second important basis for packages using Python at runtime is
whether the package in question should support multi-implementation
install or not.

A *single-impl* package is a package requiring the user to choose
exactly one Python implementation to be built against.  This means
that the scripts installed by that package will be run via specified
Python interpreter, and that the modules and extensions will be
importable from it only.  The package's Python reverse dependencies will
also have to use the same implementation.  Since the package can't
support having more than one implementation enabled, its reverse
dependencies have to be simple-impl as well.

Single-impl packages use ``python-single-r1`` eclass.  Writing ebuilds
for them is easier since it is generally sufficient to call setup
function early on, and the upstream build system generally takes care
of using selected Python version correctly.  Making packages single-impl
is recommended when dealing with packages that are not purely written
for Python or have single-impl dependencies.

A *multi-impl* package allows user to enable multiple (preferably
any number of) implementations.  The modules, extensions and scripts
installed by the package are installed separately for each enabled
implementation, and can therefore be used from any of them.  The package
can have reverse dependencies enabling only a subset of its
implementations.

Multi-impl packages use ``python-r1`` eclass.  Ebuilds are more complex
since they need to explicitly repeat build and install steps for each
enabled implementation.  Using this model is recommended for packages
providing Python modules or extensions only, or having multi-impl
reverse dependencies.  In some cases supporting multi-impl build
requires applying hacks, e.g. ``dev-libs/boost[python]`` uses
non-standard names to install ``libboost_python`` for multiple Python
versions.

The implementation for single-impl packages is selected
via ``PYTHON_SINGLE_TARGET``, while multi-impl uses ``PYTHON_TARGETS``.
These USE flag sets can be set independently to provide greater
flexibility for developers and end users.

Both single-impl and multi-impl installs are supported
by the ``distutils-r1`` eclass.


Python-first packages (distutils-r1 eclass)
===========================================
The third step in choosing the eclass for runtime use of Python
is determining whether the ebuild would benefit from ``distutils-r1``.
This eclass is especially useful for packages that primarily focus
on providing Python content.  Its advantages include:

- adding appropriate dependencies and ``REQUIRED_USE`` by default

- a sub-phase function mechanism that makes installing Python modules
  in multi-impl mode easier

- convenient support for building documentation using Sphinx
  and running tests using common Python test runners

In general, ``distutils-r1`` should be preferred over the other eclasses
if:

- the package uses a PEP 517-compliant build system (i.e. has
  a ``pyproject.toml`` file with a ``build-system`` section)

- the package uses a legacy distutils or setuptools build system
  (i.e. has a ``setup.py`` file)

- the package primarily installs Python modules

In general, for multi-impl packages ``distutils-r1`` is preferred
over ``python-r1`` as it usually makes the ebuilds simpler.
For single-impl packages, ``python-single-r1`` can sometimes be simpler.
