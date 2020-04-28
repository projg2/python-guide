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
