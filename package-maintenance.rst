==========================
Python package maintenance
==========================

.. highlight:: bash

Package name policy
===================
All packages in ``dev-python/*`` that are published on PyPI_, must be
named to match their respective PyPI names, after performing
`normalization specified in PEP 503`_, i.e. after replacing all runs
of dot, hyphen and underscore characters (``[-_.]+``) with a single
hyphen (``-``), and all uppercase letters with lowercase.  Notably,
prefixes and suffixes such as ``python-`` must not be removed or added.

Since it is not uncommon for multiple packages to be published with very
similar names, it is crucial that all packages conform to this policy.
Gentoo-specific renames not only make it harder for users to find
the specific package they need but can also create name conflicts when
another package needs to be added.  For example, adding
``python-bugzilla`` package as ``dev-python/bugzilla`` would conflict
with the ``bugzilla`` package that is also present on PyPI.

The following table illustrates mapping PyPI package names to Gentoo
package names.

  ================= ===================================================
  PyPI package name Correct Gentoo package name
  ================= ===================================================
  ConfigArgParse    ``dev-python/configargparse``
  Flask             ``dev-python/flask``
  flask-babel       ``dev-python/flask-babel``
  github3.py        ``dev-python/github3-py``
  python-bugzilla   ``dev-python/python-bugzilla``
  sphinx_pytest     ``dev-python/sphinx-pytest``
  sphinx-tabs       ``dev-python/sphinx-tabs``
  txAMQP            ``dev-python/txamqp``
  zope.interface    ``dev-python/zope-interface``
  zstd              ``dev-python/zstd``
  ================= ===================================================

Note that the presented table already presents some inconsistency
in upstream package names.  For example, ``Flask`` itself uses title
case name, while ``flask-babel`` has recently switched to lowercase.
Using normalized names for all packages avoids this inconsistency within
Gentoo.  It may also be a good idea to point upstream to `PEP 423`_
that specifies package naming recommendations.

PyPI automatically redirects to canonical package URLs.  However, please
make sure to use these canonical URLs in ``HOMEPAGE``, ``SRC_URI``
and ``remote-id`` to avoid unnecessary redirects.  These are reported
by ``pkgcheck scan --net``.

When packaging software that is not published on PyPI, or adding
multiple Gentoo packages corresponding to the same PyPI project, please
bear potential future collisions in mind when naming it.  Avoid names
that are already used for other PyPI projects.  When in doubt, prefer
more verbose names that are less likely to be reused in the future.  You
may also suggest that upstream publishes to PyPI, or at least pushes
an empty package to reserve the name.


Support for Python 2
====================
Gentoo does not support building Python packages for Python 2 anymore.
We are keeping PyPy2.7 (built stripped down, by default) to facilitate
building PyPy3, and CPython 2.7 for PyPy2.7 bootstrap.  Technically,
they could still be used to run Python 2 code standalone, but this
is discouraged and poses security risk.


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


Monitoring new package versions
===============================

PyPI release feeds
------------------
The most efficient way to follow new Python package releases are
the feeds found on PyPI_.  These can be found in the package's
"Release history" tab, as "RSS feed".

The Gentoo Python project maintains a comprehensive `list of PyPI feeds
for packages`_ in ``dev-python/`` category (as well as other important
packages maintained by the Python team) in OPML format.


Checking via pip
----------------
The `pip list -\-outdated`_ command described in a followup section
can also be used to verify installed packages against their latest PyPI
releases.  However, this is naturally limited to packages installed
on the particular system, and does not account for newer versions being
already available in the Gentoo repository.


Repology
--------
Repology_ provides a comprehensive service for tracking distribution
package versions and upstream releases.  The easiest ways to find Python
packages present in the Gentoo repository is to search by their
maintainer's e-mail or category (e.g. ``dev-python``).  When searching
by name, the majority of Python-specific package use ``python:`` prefix
in their Repology names.

Unfortunately, Repology is very susceptible to false positives.
Examples of false positives include other distributions using custom
version numbers, replacing packages with forks or simply Repology
confusing different packages with the same name.  If you find false
positives, please use the 'Report' option to request a correction.

Please also note that Repology is unable to handle the less common
version numbers that do not have a clear mapping to Gentoo version
syntax (e.g. ``.post`` releases).


Stabilization recommendations
=============================

Policy
------
The Python landscape is changing dynamically, and therefore the test
suites in packages — if not whole packages — often start failing early.
For this reason, we recommend lowering the baseline stabilization delay
to 14 days.

In addition to that:

1. When stabilizing across a major version bump (e.g. from 1.1.x
   to 1.2.x), prefer waiting until the newest minor version becomes
   the stable candidate (i.e. do not stabilize from 1.1.4 to 1.2.0
   if 1.2.1 is available).  When stabilizing over a minor version bump
   (e.g. from 1.1.4 to 1.1.5), feel free to proceed immediately.

2. If reverse dependencies block upgrade of a package (e.g. through
   ``<`` dependencies), consider stabilizing the newest versions
   matching the restriction as well.  The same is worth considering
   if upstream maintains multiple versions simultaneously with major
   API changes, even if there are no explicit ``<`` dependencies
   (e.g. ``dev-python/django``).

3. If a new release is likely to cause major compatibility issues
   (e.g. major releases of ``dev-python/sphinx``), consider delaying
   the stabilization and/or explicitly testing its reverse dependencies,
   in order to ensure that necessary ``<`` dependencies are added first.

4. Avoid stabilizing prereleases (alpha, beta and RC versions), unless
   it is necessary and upstream does not provide a final release
   for a significant time.

5. Ideally, aim for tests to pass on all relevant architectures.  Add
   deselects if necessary, as this will ensure that future
   stabilizations will be handled faster.


Tooling
-------
The recommended way of filing stabilization requests is to use
``stablereq-*`` tools from ``app-portage/mgorny-dev-scripts`` package,
combined with ``pkgdev`` from ``dev-util/pkgdev``.

To prepare the initial stabilization list and open it in an editor::

    export PKGCHECK_ARGS="--stabletime 14"
    git grep -l python@ '**/metadata.xml' |
        cut -d/ -f1-2 |
        grep -v dev-python/ |
        xargs stablereq-make-list 'dev-python/*'

Simultaneously, the following call can be used to run ``eshowkw``
to display current keywords on all stabilization candidates::

    export PKGCHECK_ARGS="--stabletime 14"
    git grep -l python@ '**/metadata.xml' |
        cut -d/ -f1-2 |
        grep -v dev-python/ |
        xargs stablereq-eshowkw 'dev-python/*'

Edit the list as desirable, save into a file and then feed the file
into pkgdev::

    pkgdev bugs --auto-cc-arches=* $(<"${file_path}")


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


.. _PyPI: https://pypi.org/

.. _normalization specified in PEP 503:
   https://peps.python.org/pep-0503/#normalized-names

.. _PEP 423: https://peps.python.org/pep-0423/

.. _list of PyPI feeds for packages:
   https://projects.gentoo.org/python/release-feeds.opml

.. _Repology: https://repology.org/
