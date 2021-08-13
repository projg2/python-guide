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

This does not apply to test dependecies — they are not strictly
necessary to install a new Portage version.
