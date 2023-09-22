==============
pytest recipes
==============

.. highlight:: bash

Skipping tests based on markers
===============================
A few packages use `custom pytest markers`_ to indicate e.g. tests
requiring Internet access.  These markers can be used to conveniently
disable whole test groups, e.g.::

    python_test() {
        epytest -m 'not network' dask
    }


Skipping tests based on paths/names
===================================
There are two primary methods of skipping tests based on path (and name)
in pytest: using ``--ignore`` and ``--deselect``.

``--ignore`` causes pytest to entirely ignore a file or a directory
when collecting tests.  This works only for skipping whole files but it
ignores missing dependencies and other failures occurring while
importing the test file.

``--deselect`` causes pytest to skip the specific test or tests.  It can
be also used to select individual tests or even parametrized variants
of tests.

Both options can be combined to get tests to pass without having
to alter the test files.  They are preferable over suggestions from
skipping problematic tests when tests are installed as part
of the package.  They can also be easily applied conditionally to Python
interpreter.

The modern versions of eclasses provide two control variables,
``EPYTEST_IGNORE`` and ``EPYTEST_DESELECT`` that can be used to list
test files or tests to be ignored or deselected respectively.  These
variables can be used in global scope to avoid redefining
``python_test()``.  However, combining them with additional conditions
requires using the local scope.

::

    python_test() {
        local EPYTEST_IGNORE=(
            # ignore whole file with missing dep
            tests/test_client.py
        )
        local EPYTEST_DESELECT=(
            # deselect a single test
            'tests/utils/test_general.py::test_filename'
            # deselect a parametrized test based on first param
            'tests/test_transport.py::test_transport_works[eventlet'
        )
        [[ ${EPYTHON} == python3.6 ]] && EPYTEST_DESELECT+=(
            # deselect a test for py3.6 only
            'tests/utils/test_contextvars.py::test_leaks[greenlet]'
        )
        epytest
    }


Avoiding the dependency on pytest-runner
========================================
pytest-runner_ is a package providing ``pytest`` command to setuptools.
While it might be convenient upstream, there is no real reason to use
it in Gentoo packages.  It has no real advantage over calling pytest
directly.

Some packages declare the dependency on ``pytest-runner``
in ``setup_requires``.  As a result, the dependency is enforced whenever
``setup.py`` is being run, even if the user has no intention of running
tests.  If this is the case, the dependency must be stripped.

The recommended method of stripping it is to use sed::

    python_prepare_all() {
        sed -i -e '/pytest-runner/d' setup.py || die
        distutils-r1_python_prepare_all
    }


.. index:: PYTEST_DISABLE_PLUGIN_AUTOLOAD
.. index:: PYTEST_PLUGINS

Disabling plugin autoloading
============================
Normally, when running a test suite pytest loads all plugins installed
on the system.  This is often convenient for upstreams, as it makes it
possible to use the features provided by the plugins (such as ``async``
test function support, or fixtures) without the necessity to explicitly
enable them.  However, there are also cases when additional plugins
could make the test suite fail or become very slow (especially if pytest
is called recursively).

The modern recommendation for these cases is to disable plugin
autoloading via setting the ``PYTEST_DISABLE_PLUGIN_AUTOLOAD``
environment variable, and then explicitly enable specific plugins
if necessary.

.. Note::

   Previously we used to recommend explicitly disabling problematic
   plugins via ``-p no:<plugin>``.  However, it is rarely obvious
   which plugin is causing the problems, and it is entirely possible
   that another plugin will cause issues in the future, so an opt-in
   approach is usually faster and more reliable.

The easier approach to enabling plugins is to use the ``-p`` option,
listing specific plugins.  The option can be passed multiple times,
and accepts a plugin name as specified in the package's
``entry_points.txt`` file::

    python_test() {
        local -x PYTEST_DISABLE_PLUGIN_AUTOLOAD=1
        epytest -p asyncio -p tornado
    }

However, this approach does not work when the test suite calls pytest
recursively (e.g. you are testing a pytest plugin).  In this case,
the ``PYTEST_PLUGINS`` environment variable can be used instead.  It
takes a comma-separated list of plugin *module names*::

    python_test() {
        local -x PYTEST_DISABLE_PLUGIN_AUTOLOAD=1
        local -x PYTEST_PLUGINS=xdist.plugin,xdist.looponfail,pytest_forked

        epytest
    }

Please note that failing to enable all the required plugins may cause
some of the tests to be skipped implicitly (especially if the test suite
is using ``async`` functions and no async plugin is loaded).  Please
look at skip messages and warnings to make sure everything works
as intended.


Using pytest-xdist to run tests in parallel
===========================================
pytest-xdist_ is a plugin that makes it possible to run multiple tests
in parallel.  This is especially useful for programs with large test
suites that take significant time to run single-threaded.

Not all test suites support pytest-xdist.  Particularly, it requires
that the tests are written not to collide one with another.  Sometimes,
xdist may also cause instability of individual tests.

When only a few tests are broken or unstable because of pytest-xdist,
it is possible to use it and deselect the problematic tests.  It is up
to the maintainer's discretion to decide whether this is justified.

Using pytest-xdist is recommended if the package in question supports it
(i.e. it does not cause semi-random test failures) and its test suite
takes significant time.  When using pytest-xdist, please respect user's
make options for the job number, e.g.::

    inherit multiprocessing

    BDEPEND="
        test? (
            dev-python/pytest-xdist[${PYTHON_USEDEP}]
        )
    "

    distutils_enable_tests pytest

    python_test() {
       epytest -n "$(makeopts_jobs)" --dist=worksteal
    }

Please note that some upstream use pytest-xdist even if there is no real
gain from doing so.  If the package's tests take a short time to finish,
please avoid the dependency and strip it if necessary.

The ``--dist=worksteal`` enables rescheduling tests when some of
the workers finish early.  It is recommended when some of the package's
tests are very slow while others are fast.  Otherwise, the lengthy tests
may end up being executed on the same thread and become a bottleneck.


Dealing with flaky tests
========================
A flaky test is a test that sometimes passes, and sometimes fails
with a false positive result.  Often tests are flaky because of too
steep timing requirements or race conditions.  While generally it is
preferable to fix the underlying issue (e.g. by increasing timeouts),
it is not always easy.

Sometimes upstreams use such packages as ``dev-python/flaky``
or ``dev-python/pytest-rerunfailures`` to mark tests as flaky and have
them rerun a few minutes automatically.  If upstream does not do that,
it is also possible to force a similar behavior locally in the ebuild::

    python_test() {
        # plugins make tests slower, and more fragile
        local -x PYTEST_DISABLE_PLUGIN_AUTOLOAD=1
        # some tests are very fragile to timing
        epytest -p rerunfailures --reruns=10 --reruns-delay=2
    }

Note that the snippet above also disables plugin autoloading to speed
tests up and therefore reduce their flakiness.  Sometimes forcing
explicit rerun also makes it possible to use xdist on packages that
otherwise randomly fail with it.


Avoiding dependencies on other pytest plugins
=============================================
There is a number of pytest plugins that have little value to Gentoo
users.  They include plugins for test coverage
(``dev-python/pytest-cov``), coding style (``dev-python/pytest-flake8``)
and more.  Generally, packages should avoid using those plugins.

.. Warning::

   As of 2022-01-24, ``epytest`` disables a few undesirable plugins
   by default.  As a result, developers have a good chance
   of experiencing failures due to hardcoded pytest options first,
   even if they have the relevant plugins installed.

   If your package *really* needs to use the specific plugin, you need
   to pass ``-p <plugin>`` explicitly to reenable it.

In some cases, upstream packages only list them as dependencies
but do not use them automatically.  In other cases, you will need
to strip options enabling them from ``pytest.ini`` or ``setup.cfg``.

::

    src_prepare() {
        sed -i -e 's:--cov=wheel::' setup.cfg || die
        distutils-r1_src_prepare
    }


TypeError: _make_test_flaky() got an unexpected keyword argument 'reruns'
=========================================================================
If you see a test error resembling the following::

    TypeError: _make_test_flaky() got an unexpected keyword argument 'reruns'

This means that the tests are being run via flaky_ plugin while
the package in question expects pytest-rerunfailures_.  This is
because both plugins utilize the same ``@pytest.mark.flaky`` marker
but support different set of arguments.

To resolve the problem, explicitly disable the ``flaky`` plugin and make
sure to depend on ``dev-python/pytest-rerunfailures``::

    BDEPEND="
        test? (
             dev-python/pytest-rerunfailures[${PYTHON_USEDEP}]
        )"

    python_test() {
        epytest -p no:flaky
    }


ImportPathMismatchError
=======================
An ``ImportPathMismatchError`` generally indicates that the same Python
module (or one that supposedly looks the same) has been loaded twice
using different paths, e.g.::

    E   _pytest.pathlib.ImportPathMismatchError: ('path', '/usr/lib/pypy3.7/site-packages/path', PosixPath('/tmp/portage/dev-python/jaraco-path-3.3.1/work/jaraco.path-3.3.1/jaraco/path.py'))

These problems are usually caused by pytest test discovery getting
confused by namespace packages.  In this case, the ``jaraco`` directory
is a Python 3-style namespace but pytest is treating it as a potential
test directory.  Therefore, instead of loading it as ``jaraco.path``
relatively to the top directory, it loads it as ``path`` relatively
to the ``jaraco`` directory.

The simplest way to resolve this problem is to restrict the test
discovery to the actual test directories, e.g.::

    python_test() {
        epytest test
    }

or::

    python_test() {
        epytest --ignore jaraco
    }


fixture '...' not found
=======================
Most of the time, a missing fixture indicates that some pytest plugin
is not installed.  In rare cases, it can signify an incompatible pytest
version or package issue.

The following table maps common fixture names to their respective
plugins.

=================================== ====================================
Fixture name                        Package
=================================== ====================================
event_loop                          dev-python/pytest-asyncio
freezer                             dev-python/pytest-freezegun
httpbin                             dev-python/pytest-httpbin
loop                                dev-python/pytest-aiohttp
mocker                              dev-python/pytest-mock
=================================== ====================================


Warnings
========
pytest captures all warnings from the test suite by default, and prints
a summary of them at the end of the test suite run::

    =============================== warnings summary ===============================
    asgiref/sync.py:135: 1 warning
    tests/test_local.py: 5 warnings
    tests/test_sync.py: 12 warnings
    tests/test_sync_contextvars.py: 1 warning
      /tmp/asgiref/asgiref/sync.py:135: DeprecationWarning: There is no current event loop
        self.main_event_loop = asyncio.get_event_loop()
    [...]

However, some projects go further and use ``filterwarnings`` option
to make (some) warnings fatal::

    ==================================== ERRORS ====================================
    _____________________ ERROR collecting tests/test_sync.py ______________________
    tests/test_sync.py:577: in <module>
        class ASGITest(TestCase):
    tests/test_sync.py:583: in ASGITest
        async def test_wrapped_case_is_collected(self):
    asgiref/sync.py:135: in __init__
        self.main_event_loop = asyncio.get_event_loop()
    E   DeprecationWarning: There is no current event loop
    =========================== short test summary info ============================
    ERROR tests/test_sync.py - DeprecationWarning: There is no current event loop
    !!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
    =============================== 1 error in 0.13s ===============================

Unfortunately, this frequently means that warnings coming from
a dependency trigger test failures in other packages.  Since making
warnings fatal is relatively common in the Python world, it is
recommended to:

1. Fix warnings in Python packages whenever possible, even if they
   are not fatal to the package itself.

2. Do not enable new Python implementations if they trigger any new
   warnings in the package.

If the warnings come from issues in the package's test suite rather than
the installed code, it is acceptable to make them non-fatal.  This can
be done either through removing the ``filterwarnings`` key from
``setup.cfg``, or adding an ignore entry.  For example, the following
setting ignores ``DeprecationWarning`` in ``test`` directory::

    filterwarnings =
        error
        ignore::DeprecationWarning:test


.. _custom pytest markers:
   https://docs.pytest.org/en/stable/example/markers.html
.. _pytest-runner: https://pypi.org/project/pytest-runner/
.. _pytest-xdist: https://pypi.org/project/pytest-xdist/
.. _flaky: https://github.com/box/flaky/
.. _pytest-rerunfailures:
   https://github.com/pytest-dev/pytest-rerunfailures/
