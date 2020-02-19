================================
Choosing between Python eclasses
================================

Build-time vs runtime use
=========================
The first basis for choosing Python eclass is whether Python is used
merely at build-time or at runtime as well.

A runtime use occurs if the package explicitly needs Python to be
installed along with it, in order for it to function correctly.  This
generally happens if the package installs Python modules, extensions,
scripts, or executables calling the Python interpreter or linking
to libpython.  This also applies to bash scripts or other executables
that call python inline.

A build-time use occurs if the package calls the Python interpreter
or any kind of aforementioned executables during package's build
(or install) phases.

If the package uses Python purely at build-time, the ``python-any-r1``
eclass is appropriate.  Otherwise, ``python-single-r1``, ``python-r1``
or their derivatives are to be used.

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
requieres applying hacks, e.g. ``dev-libs/boost[python]`` uses
non-standard names to install ``libboost_python`` for multiple Python
versions.

The implementation for single-impl packages is selected
via ``PYTHON_SINGLE_TARGET``, while multi-impl uses ``PYTHON_TARGETS``.
These USE flag sets can be set independently to provide greater
flexibility for developers and end users.


Distutils and related build systems
===================================
The third basis for choosing an eclass is the build system used.
If the project uses one of Python-specific build systems, that is
distutils, setuptools, flit or poetry, the ``distutils-r1`` eclass
should be used instead of the other eclasses.  As a rule of thumb,
this happens when either ``setup.py`` or ``pyproject.toml`` file exists
in the distribution.

``distutils-r1`` builds on either ``python-r1`` or ``python-single-r1``,
therefore it can be used to create both multi-impl and single-impl
packages.  It provides full set of default phase functions, making
writing ebuilds much easier.


A rule of thumb
===============
As a rule of thumb, the following checklist can be used to determine
the eclass to use:

1. If the package has ``setup.py`` or ``pyproject.toml`` file,
   use ``distutils-r1``.

2. If the package primarily installs Python modules or extensions
   or has multi-impl reverse dependencies, use ``python-r1``.

3. If the package (possibly conditionally) qualifies as using Python
   at runtime, use ``python-single-r1``.

4. If the package uses Python at build time only, use ``python-any-r1``.
