===================
Python interpreters
===================

Versions of Python
==================
By a *version of Python* we usually mean the variant of Python language
and standard library interface as used by a specific version
of CPython_, the reference implementation of Python.

Python versions are determined from the two first version components.
The major version is incremented when major incompatible changes are
introduced in the language, as was the case in Python 3.  Along with
minor version changes, the new releases introduce new features
and remove deprecated APIs.  The Python documentation generally
indicates when a particular API was added or deprecated, and when it
is planned to be removed.

Practically speaking, this means that a program written purely
for Python 2 is unlikely to work on Python 3, and requires major changes
to achieve compatibility.  On the other hand, a program written for
Python 3.7 is very likely to work with Python 3.8, and reasonably likely
to support Python 3.6 as well.  If that is not the case, minor changes
are usually sufficient to fix that.

For example, Python 3.7 introduced a new `importlib.resources`_ module.
If your program uses it, it will not work on Python 3.6 without
a backwards compatibility code.

Python 3.8 removed the deprecated `platform.linux_distribution()`_
function.  If your program used it, it will not work on Python 3.8
without changes.  However, it was deprecated since Python 3.5, so if you
were targetting 3.7, you should not have been using it in the first
place.

Gentoo supports building packages against Python 2.7 and a shifting
window of 3-4 versions of Python 3.  They are provided as slots
of ``dev-lang/python``.


Life cycle of a Python implementation
=====================================
Every Python implementation (understood as a potential target) in Gentoo
follows roughly the following life cycle:

1. The interpreter is added to ``~arch`` for initial testing.  At this
   point, packages can not declare support for the implementation yet.

   New CPython releases enter this stage when upstream releases
   the first alpha version.  Since the feature set is not stable yet,
   it is premature to declare the package compatibility with these
   versions.

2. The new Python target is added.  It is initially stable-masked,
   so only ``~arch`` users can use it.  At this point, packages start
   being tested against the new target and its support starts being
   declared in ``PYTHON_COMPAT``.

   CPython releases enter this stage when the first beta release
   is made.  These versions are not fully stable yet either and package
   regressions do happen between beta releases but they are stable
   enough for initial testing.

3. When ready, the new interpreter is stabilized.  The target is not yet
   available for stable users, though.

   CPython releases enter this stage roughly 30 days after the first
   stable release (i.e. X.Y.0 final) is made.  This is also the stage
   where PyPy3 releases are in Gentoo for the time being.

4. The stable-mask for the target is removed.  For this to happen,
   the inconsistencies in stable graph need to be addressed first
   via stabilizing newer versions of packages.

   CPython releases enter this stage after the interpreter is marked
   stable on all architectures and all the packages needed to clear
   the stable depenendency graph are stable as well.

5. Over time, developers are repeatedly asked to push testing packages
   for the new target forward and stabilize new versions supporting it.
   Eventually, the final push for updates happens and packages
   not supporting the new target start being removed.

6. If applicable, the new target becomes the default.  The developers
   are required to test new packages against it.  The support for old
   target is slowly being discontinued.

   Currently, we try to plan the switch to the next CPython version
   around June — July every year, to make these events more predictable
   to Gentoo users.

7. Eventually, the target becomes replaced by the next one.  When it
   nears end of life, the final packages requiring it are masked for
   removal and the target flags are disabled.

   We generally aim to preserve support for old targets for as long
   as they are found still needed by Gentoo users.  However, as more
   upstream packages remove support for older versions of Python,
   the cost of preserving the support becomes too great.

8. The compatibility declarations are cleaned up from ``PYTHON_COMPAT``,
   and obsolete ebuild and eclass code is cleaned up.

9. Finally, the interpreter is removed when it becomes no longer
   feasible to maintain it (usually because of the cost of fixing
   vulnerabilities or build failures).


Stability guarantees of Python implementations
==============================================
The language and standard library API of every Python version is
expected to be stable since the first beta release of the matching
CPython version.  However, historically there were cases of breaking
changes prior to a final release (e.g. the revert of ``enum`` changes
in Python 3.10), as well as across minor releases (e.g. ``urlsplit()``
URL sanitization / security fix).

The ABI of every CPython version is considered stable across bugfix
releases since the first RC.  This includes the public ABI of libpython,
C extensions and compiled Python modules.  Prior to the first RC,
breaking changes to either may still happen.  Gentoo currently does not
account for these changes to the high cost of using slot operators,
and therefore users using ``~arch`` CPython may have to occasionally
rebuild Python packages manually.

Additionally, modern versions of CPython declare so-called 'stable ABI'
that remains forward compatible across Python versions.  This permits
upstreams to release wheels that can be used with multiple CPython
versions (contrary to the usual case of building wheels separately
for each version).  However, this does not affect Gentoo packaging
at the moment.

PyPy does not hold specific ABI stability guarantees.  Gentoo packages
use subslots to declare the current ABI version, and the eclasses use
slot operators in dependencies to enforce rebuilds whenever the ABI
version changes.  Fortunately, this has not happened recently.


Alternative Python implementations
==================================
CPython is the reference and most commonly used Python implementation.
However, there are other interpreters that aim to maintain reasonable
compatibility with it.

PyPy_ is an implementation of Python built using in-house RPython
language, using a Just-in-Time compiler to achieve better performance
(generally in long-running programs running a lot of Python code).
It maintains quite good compatibility with CPython, except when programs
rely on its implementation details or GC behavior.

PyPy upstream provides PyPy variants compatible with Python 2.7
and at least one version of Python 3.  Gentoo supports building packages
against the current version of PyPy3 (and the previous version during
transition periods).  The different versions are available as slots
of ``dev-lang/pypy``.

Jython_ is an implementation of Python written in Java.  Besides being
a stand-alone Python interpreter, it supports bidirectional interaction
between Python and Java libraries.

Jython development is very slow paced, and it is currently bound
to Python 2.7.  Gentoo does not provide Jython anymore.

IronPython_ is an implementation of Python for the .NET framework.
Alike Jython, it supports bidirectional interaction between Python
and .NET Framework.  It is currently bound to Python 2.7.  It is not
packaged in Gentoo.

Brython_ is an implementation of Python 3 for client-side web
programming (in JavaScript).  It provides a subset of Python 3 standard
library combined with access to DOM objects.

MicroPython_ is an implementation of Python 3 aimed for microcontrollers
and embedded environments.  It aims to maintain some compatibility
with CPython while providing stripped down standard library
and additional modules to interface with hardware.  It is packaged
as ``dev-lang/micropython``.

Tauthon_ is a fork of Python 2.7 that aims to backport new language
features and standard library modules while preserving backwards
compatibility with existing code.  It is not packaged in Gentoo.


Support for multiple implementations
====================================
The support for simultaneously using multiple Python implementations
is implemented primarily through USE flags.  The packages installing
or using Python files define either ``PYTHON_TARGETS``
or ``PYTHON_SINGLE_TARGET`` flags that permit user to choose which
implementations are used.

Modules and extensions are installed separately for each interpreter,
in its specific site-packages directory.  This means that a package
can run using a specific target correctly only if all its dependencies
were also installed for the same implementation.  This is enforced
via USE dependencies.

Additionally, ``dev-lang/python-exec`` provides a mechanism for
installing multiple variants of each Python script simultaneously.  This
is necessary to support scripts that differ between Python versions
(particularly between Python 2 and Python 3) but it is also used
to prevent scripts from being called via unsupported interpreter
(i.e.  one that does not have its accompanying modules or dependencies
installed).

This also implies that all installed Python scripts must have their
shebangs adjusted to use a specific Python interpreter (not ``python``
nor ``python3`` but e.g. ``python3.7``), and all other executables must
also be modified to call specific version of Python directly.


Implementation support policy
=============================
Gentoo gives the following guarantees with regards to Python support:

1. We aim to provide support for new CPython targets as soon as that
   becomes possible, and switch the default to the new slot as soon
   as porting is considered reasonably complete.  This usually happens
   around June, the next year after the first stable release on a given
   branch.

2. We will continue providing support for the previous CPython branch
   for at least one year after the next branch becomes stable.  However,
   in reality this will probably extend until PyPy switches to the newer
   slot, as we aim to support parallel versions of PyPy and CPython.

3. We will continue providing interpreters (possibly without package
   target support) for as long as they are maintained upstream, i.e.
   until their end-of-life date.  Afterwards, we may continue providing
   EOL interpreters for as long as patching them continues being
   feasible.

4. Experimental targets (e.g. freethreading CPython, PyPy) are provided
   on best-effort basis, with no support guarantees.


Backports
=========
A common method of improving compatibility with older versions of Python
is to backport new standard library modules or features.  Packages doing
that are generally called *backports*.

Ideally, backports copy the code from the standard library with minimal
changes, and provide a matching API.  In some cases, new versions
of backports are released as the standard library changes, and their
usability extends from providing a missing module to extending older
version of the module.  For example, the ``dev-python/funcsigs`` package
originally backported function signatures from Python 3.3 to older
versions, and afterwards was updated to backport new features from
Python 3.6, becoming useful to versions 3.3 through 3.5.

Sometimes, the opposite happens.  ``dev-python/mock`` started
as a stand-alone package, and was integrated into the standard library
as unittest.mock_ later on.  Afterwards, the external package became
a backport of the standard library module.

In some cases backports effectively replace external packages.  Once
lzma_ module has been added to the standard library, its backport
``dev-python/backports-lzma`` has effectively replaced the competing
LZMA packages.

Individual backports differ by the level of compatibility with
the standard library provided, and therefore on the amount of additional
code needed in your program.  The exact kind of dependencies used
depends on that.

``dev-python/ipaddress`` is a drop-in backport of the ipaddress_ module
from Python 3.3.  It is using the same module name, so a code written
to use this module will work out-of-the-box on Python 2.7 if the package
is installed.  As a side note, since Python always prefers built-in
modules over external packages, there is no point in enabling Python 3
in this package as the installed module would never be used.
Appropriately, you should depend on this package only for the Python
versions needing it.

``dev-python/mock`` is a compatible backport of the unittest.mock_
module.  It can't use the same name as the standard library module,
therefore the packages need to use it conditionally, e.g.::

    try:
        from unittest.mock import Mock
    except ImportError:  # py<3.3
        from mock import Mock

or::

    import sys
    if sys.hexversion >= 0x03030000:
        from unittest.mock import Mock
    else:
        from mock import Mock

However, the actual API remains compatible, so the programs do not need
more compatibility code than that.  In some cases, upstreams fail (or
even refuse) to use the external ``mock`` package conditionally —
in that case, you either need to depend on this package unconditionally,
or patch it.

``dev-python/trollius`` aimed to provide a backport of asyncio_
for Python 2.  Since the asyncio framework relies on new Python syntax,
the backport cannot be API compatible and requires using a different
syntax than native asyncio code.


.. _CPython: https://www.python.org/

.. _importlib.resources:
   https://docs.python.org/3.7/library/importlib.html#module-importlib.resources

.. _platform.linux_distribution():
   https://docs.python.org/3.7/library/platform.html#platform.linux_distribution

.. _PyPy: https://www.pypy.org/

.. _Jython: https://www.jython.org/

.. _IronPython: https://ironpython.net/

.. _Brython: https://www.brython.info/

.. _MicroPython: https://micropython.org/

.. _Tauthon: https://github.com/naftaliharris/tauthon

.. _unittest.mock:
   https://docs.python.org/3.3/library/unittest.mock.html

.. _lzma: https://docs.python.org/3.3/library/lzma.html

.. _ipaddress: https://docs.python.org/3.3/library/ipaddress.html

.. _asyncio: https://docs.python.org/3.4/library/asyncio.html
