==========================
Python package maintenance
==========================

Support for Python 2
====================
Since Python 2.7 reached EOL, Gentoo is currently phasing out support
for Python 2.  Unless your package or its reverse dependencies really
need it, you should omit it from ``PYTHON_COMPAT``.  If you're adding
a new package and it does not support Python 3, do not add it.

Many upstreams are removing Python 2 support from new releases of their
software.  We remove it proactively whenever reverse dependencies permit
in order to anticipate this and avoid having to deal with lots
of reverse dependencies afterwards.

Packages that do not support Python 3 and are unlikely to start
supporting it soon are being slowly removed.


Which implementations to test new packages for?
===============================================
The absolute minimum set of targets are the current default targets
found in ``profiles/base/make.defaults``.  However, developers
are strongly encouraged to test at least the next Python 3 version
in order to ease future transition, and preferably all future versions.

Marking for PyPy3 is optional.  At this moment, we do not aim for wide
coverage of PyPy3 support.


Adding new Python implementations to existing packages
======================================================
New Python implementations can generally be added to existing packages
without a revision bump.  This is because the new dependencies are added
conditionally to new USE flags.  Since the existing users can not have
the new flags enabled, the dependencies do not need to be proactively
added to existing installations.

This usually applies to stable packages as well as new Python targets
are generally ``use.stable.mask``-ed.  This means that stable users
will not be able to enable newly added flags and therefore the risk
of the change breaking stable systems is minimal.


Which packages can be (co-)maintained by the Python project?
============================================================
A large part of the Python ecosystem is fairly consistent, making it
feasible for (co-)maintenance by the Gentoo Python team.

As a rule of thumb, Python team is ready to maintain packages specific
to the Python ecosystem and useful for the general population of Python
programmers.  This includes Python interpreters and tooling, packages
purely providing Python modules and extensions and utilities specific
to the Python language.

However, the Python team has limited manpower, therefore it may reject
packages that have high maintenance requirements.  As a rule, Python
team does not accept packages without working tests.

If your package matches the above profile, feel free to ask a member
of the Python project whether they would like to (co-)maintain
the package.  However, if you are not a member of the project, please
do not add us without asking first.


Porting packages to a new EAPI
==============================
When porting packages to a new EAPI, please take care not to port
the dependencies of Portage prematurely.  This generally includes
``app-portage/gemato``, ``dev-python/setuptools`` and their recursive
dependencies.

Ideally, these ebuilds carry an appropriate note above their EAPI line,
e.g.::

    # please keep this ebuild at EAPI 7 -- sys-apps/portage dep
    EAPI=7

This does not apply to test dependencies — they are not strictly
necessary to install a new Portage version.


Routine checks on installed Python packages
===========================================
The following actions are recommended to be run periodically on systems
used to test Python packages.  They could be run e.g. via post-sync
actions.


pip check
---------
``pip check`` (provided by ``dev-python/pip``) can be used to check
installed packages for missing dependencies and version conflicts:

.. code-block:: text

    $ python3.10 -m pip check
    meson-python 0.6.0 requires ninja, which is not installed.
    cx-freeze 6.11.1 requires patchelf, which is not installed.
    openapi-spec-validator 0.4.0 has requirement openapi-schema-validator<0.3.0,>=0.2.0, but you have openapi-schema-validator 0.3.0.
    cx-freeze 6.11.1 has requirement setuptools<=60.10.0,>=59.0.1, but you have setuptools 62.6.0.

This tool checks the installed packages for a single Python
implementation only, so you need to run it for every installed
interpreter separately.

In some cases the issues are caused by unnecessary version pins
or upstream packages listing optional dependencies as obligatory.
The preferred fix is to fix the package metadata rather than modifying
the dependencies in ebuild.

.. Warning::

   pip does not support the ``Provides`` metadata, so it can
   produce false positives about ``certifi`` dependency.  Please ignore
   these:

   .. code-block:: text

       httpcore 0.15.0 requires certifi, which is not installed.
       httpx 0.23.0 requires certifi, which is not installed.
       sphobjinv 2.2.2 requires certifi, which is not installed.
       requests 2.28.0 requires certifi, which is not installed.


pip list -\-outdated
--------------------
``pip list --outdated`` (provided by ``dev-python/pip``) can be used
to check whether installed packages are up-to-date.  This can help
checking for pending version bumps, as well as to detect wrong versions
in installed metadata:

.. code-block:: text

    $ pip3.11 list --outdated
    Package                  Version           Latest  Type
    ------------------------ ----------------- ------- -----
    dirty-equals             0                 0.4     wheel
    filetype                 1.0.10            1.0.13  wheel
    mercurial                6.1.3             6.1.4   sdist
    node-semver              0.8.0             0.8.1   wheel
    PyQt-builder             1.12.2            1.13.0  wheel
    PyQt5                    5.15.6            5.15.7  wheel
    PyQt5-sip                12.10.1           12.11.0 sdist
    PyQtWebEngine            5.15.5            5.15.6  wheel
    Routes                   2.5.1.dev20220522 2.5.1   wheel
    selenium                 3.141.0           4.3.0   wheel
    sip                      6.6.1             6.6.2   wheel
    sphinxcontrib-websupport 1.2.4.dev20220515 1.2.4   wheel
    uri-template             0.0.0             1.2.0   wheel
    watchfiles               0.0.0             0.15.0  wheel
    watchgod                 0.0.dev0          0.8.2   wheel

Again, the action applies to a single Python implementation only
and needs to be repeated for all of them.

Particularly note the packages with versions containing only zeroes
in the above list — this is usually a sign that the build system
does not recognize the version correctly.  In some cases, the only
working solution would be to sed the correct version in.

The additional ``dev`` suffix is usually appended via ``tag_build``
option in ``setup.cfg``.  This causes the version to be considered
older than the actual release, and therefore the respective options need
to be stripped.


gpy-verify-deps
---------------
``gpy-verify-deps`` (provided by ``app-portage/gpyutils``) compares
the ebuild dependencies of all installed Python packages against their
metadata.  It reports the dependencies that are potentially missing
in ebuilds, as well as dependencies potentially missing
``[${PYTHON_USEDEP}]``.  For the latter, it assumes that all
dependencies listed in package metadata are used as Python modules.

.. code-block:: text

    $ gpy-verify-deps
    [...]
    =dev-python/tempest-31.0.0: missing dependency: dev-python/oslo-serialization [*]
    =dev-python/tempest-31.0.0: missing dependency: dev-python/cryptography [*]
    =dev-python/tempest-31.0.0: missing dependency: dev-python/stestr [*]
    =dev-python/versioningit-2.0.0: missing dependency: dev-python/tomli [*]
    =dev-python/versioningit-2.0.0: missing dependency: dev-python/importlib_metadata [python3.8 python3.9]
    =dev-python/wstools-0.4.10-r1: missing dependency: dev-python/setuptools [*]

The check is done for all installed interpreters.  The report indicates
whether the dependency upstream is unconditional (``[*]``) or specific
to a subset of Python implementations.

Similarly to ``pip check`` results, every dependency needs to be
verified.  In many cases, upstream metadata lists optional or build-time
dependencies as runtime dependencies, and it is preferable to strip them
than to copy the mistakes into the ebuild.
