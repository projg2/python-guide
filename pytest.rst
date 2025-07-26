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


.. index:: EPYTEST_DESELECT
.. index:: EPYTEST_IGNORE

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
.. index:: EPYTEST_PLUGINS
.. index:: EPYTEST_PLUGIN_AUTOLOAD
.. index:: EPYTEST_PLUGIN_LOAD_VIA_ENV

Controlling pytest plugins used
===============================
pytest supports plugins that can extend the functionality of test
suites.  Plugin packages range from defining reusable fixtures
to severely altering pytest behavior.  By default, pytest automatically
loads all plugins found in the environment.  This is a good default
for isolated test environments where the available plugins are strictly
controlled.  However, in Gentoo test suites are run against the system
Python install which can feature a large number of different pytest
plugins installed.  These unexpected plugins can lead to results ranging
from the test run becoming unnecessarily slow to failing in confusing
ways.

For this reason, it is recommended to explicitly control pytest plugins
used.  To achieve this, ``EPYTEST_PLUGINS`` array can be specified.
Specifying the variable always disables plugin autoloading.  When one
or more package names (without category) are specified,
``distutils_enable_tests`` adds dependencies on these packages
and ``epytest`` adds appropriate ``-p`` arguments to load their entry
points.  Conversely, if the variable is set to an empty array,
no plugins are loaded.

::

    # disable plugin autoloading
    EPYTEST_PLUGINS=()
    distutils_enable_tests pytest

    # add dependencies and load plugins
    EPYTEST_PLUGINS=( pytest-asyncio pytest-mock )
    distutils_enable_tests pytest

.. Note::

   Historically, we used to specify ``PYTEST_DISABLE_PLUGIN_AUTOLOAD``
   explicitly in ebuilds.  This is done automatically
   by ``EPYTEST_PLUGINS``, and therefore explicit exports can be removed
   after transitioning to it.

Some plugins require additional arguments to actually become effective.
If these arguments are not specified in the upstream configuration file,
you may need to override ``python_test()`` and specify them explicitly,
e.g.::

    EPYTEST_PLUGINS=( pytest-{asyncio,forked,mock} )
    distutils_enable_tests pytest

    python_test() {
        # --forked to workaround protobuf segfaults
        # https://github.com/protocolbuffers/protobuf/issues/22067
        epytest --forked
    }

Plugins that are enabled via other ``EPYTEST_*`` variables do not need
to be repeated in ``EPYTEST_PLUGINS``.

While ``EPYTEST_PLUGINS`` aims to support the most common use cases,
it is not sufficient for all test suites.  In particular, test suites
for pytest plugins often rely on the plugins being loaded implicitly
in a subprocess.  In these cases, ``EPYTEST_PLUGIN_LOAD_VIA_ENV``
can be used.  It tells the eclass to set the ``PYTEST_PLUGINS``
environment variable which is respected by subprocesses.  Note that
it is permitted to use ``${PN}`` in ``EPYTEST_PLUGINS`` — the eclass
will not add a self-dependency in that case::

    # xdist is also used in subtests
    EPYTEST_PLUGINS=( "${PN}" pytest-{rerunfailures,xdist} )
    EPYTEST_PLUGIN_LOAD_VIA_ENV=1
    EPYTEST_XDIST=1
    distutils_enable_tests pytest

In some cases, it is very hard to get the test suite working correctly
with plugin autoloading.  In these cases, ``EPYTEST_PLUGIN_AUTOLOAD``
variable can be used to explicitly specify that autoloading is
desirable.  This variable can be combined with ``EPYTEST_PLUGINS``,
in which case the eclass will still automatically add the dependencies::

    EPYTEST_PLUGINS=( pytest-asyncio )
    EPYTEST_PLUGIN_AUTOLOAD=1
    distutils_enable_tests pytest


.. index:: EPYTEST_XDIST
.. index:: pytest-xdist

Using pytest-xdist to run tests in parallel
===========================================
pytest-xdist_ is a plugin that makes it possible to run multiple tests
in parallel.  This is especially useful for programs with large test
suites that take significant time to run single-threaded.

Using pytest-xdist is recommended if the package in question supports it
(i.e. it does not cause semi-random test failures) and its test suite
takes significant time.  This is done via setting ``EPYTEST_XDIST``
to a non-empty value prior to calling ``distutils_enable_tests``.
It ensures that an appropriate depedency is added, and that ``epytest``
adds necessary command-line options.

.. code-block::

    EPYTEST_XDIST=1
    distutils_enable_tests pytest

Please note that some upstream use pytest-xdist even if there is no real
gain from doing so.  If the package's tests take a short time to finish,
please avoid the dependency and strip it if necessary.

Not all test suites support pytest-xdist.  Particularly, it requires
that the tests are written not to collide one with another.  Sometimes,
xdist may also cause instability of individual tests.  In some cases,
it is possible to work around this using the same solution as when
`dealing with flaky tests`_.

When only a few tests are broken or unstable because of pytest-xdist,
it is possible to use it and deselect the problematic tests.  It is up
to the maintainer's discretion to decide whether this is justified.


.. index:: flaky
.. index:: pytest-rerunfailures

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

    EPYTEST_PLUGINS=( pytest-rerunfailures )
    distutils_enable_tests pytest

    python_test() {
        # some tests are very fragile to timing
        epytest --reruns=10 --reruns-delay=2
    }

Note that the snippet above also disables plugin autoloading to speed
tests up and therefore reduce their flakiness.  Sometimes forcing
explicit rerun also makes it possible to use xdist on packages that
otherwise randomly fail with it.


.. index:: EPYTEST_TIMEOUT
.. index:: pytest-timeout

Using pytest-timeout to prevent deadlocks (hangs)
=================================================
pytest-timeout_ plugin adds an option to terminate the test if its
runtime exceeds the specified limit.  Some packages decorate specific
tests with timeouts; however, it is also possible to set a baseline
timeout for all tests.

A timeout causes the test run to fail, and therefore using it is
not generally necessary for test suites that are working correctly.
If individual tests are known to suffer from unfixable hangs, it is
preferable to deselect them.  However, setting a general timeout is
recommended when a package is particularly fragile, or has suffered
deadlocks in the past.  A proactive setting can prevent it from hanging
and blocking arch testing machines.

The plugin can be enabled via setting ``EPYTEST_TIMEOUT`` to the timeout
in seconds, prior to calling ``distutils_enable_tests``.  This ensures
that an appropriate depedency is added, and that ``epytest`` adds
necessary command-line options.

.. code-block::

    : ${EPYTEST_TIMEOUT:=1800}
    distutils_enable_tests pytest

The timeout applies to every test separately, i.e. the above example
will cause a single test to time out after 30 minutes.  If multiple
tests hang, the total run time will multiply consequently.

When deciding on a timeout value, please take into the consideration
that the tests may be run on a low performance hardware, and on a busy
system, and choose an appropriately high value.

It is a good idea to use the default assignment form, as in the snippet
above, as that permits the user to easily override the timeout
if necessary.

.. Note::

   ``EPYTEST_TIMEOUT`` can also be set by user in ``make.conf``
   or in the calling environment.  This can be used as a general
   protection against hanging test suites.  However, please note that
   this does not control dependencies, and therefore the user may need
   to install ``dev-python/pytest-timeout`` explicitly.


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
   to explicitly enable it via ``PYTEST_PLUGINS``.

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

To resolve the problem, explicitly use
``dev-python/pytest-rerunfailures``::

    EPYTEST_PLUGINS=( pytest-rerunfailures )
    distutils_enable_tests pytest


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


Failures due to missing files in temporary directories
======================================================
As of 2024-01-05, ``epytest`` overrides the default temporary directory
retention policy of pytest.  By default, directories from successful
tests are removed immediately, and the temporary directories
from the previous test run are replaced by the subsequent test run.
This frequently reduces disk space requirements from test suites,
but it can rarely cause tests to fail.

If you notice test failures combined with indications that a file was
not found, and especially regarding the pytest temporary directories,
try if overriding the retention policy helps, e.g.::

    python_test() {
        epytest -o tmp_path_retention_policy=all
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
event_loop                          dev-python/pytest-asyncio (<2)
freezer                             dev-python/pytest-freezegun
httpbin                             dev-python/pytest-httpbin
loop                                dev-python/pytest-aiohttp
mocker                              dev-python/pytest-mock
=================================== ====================================


.. index:: filterwarnings
.. index:: Werror
.. index:: pytest.warns

Warnings and ``pytest.raises()``
================================
Some projects set pytest to raise on warnings, using options such as:

.. code-block:: toml

    filterwarnings = [
        "error",
        # ...
    ]

This may be desirable for upstream CI systems, as it ensures that pull
requests do not introduce new warnings, and that any deprecations are
handled promptly.  However, it is undesirable for downstream testing,
as new deprecations in dependencies can cause the existing versions
to start failing.

To avoid this problem, ``epytest`` explicitly forces ``-Wdefault``.
Most of the time, this does not cause any issues, besides causing pytest
to verbosely report warnings that are normally ignored by the test
suite.  However, if some packages incorrectly use ``pytest.raises()``
to check for warnings, their test suites will fail, for example::

    ============================= FAILURES =============================
    ________________ test_ser_ip_with_unexpected_value _________________

        def test_ser_ip_with_unexpected_value() -> None:
            ta = TypeAdapter(ipaddress.IPv4Address)

    >       with pytest.raises(UserWarning, match='serialized value may not be as expected.'):
    E       Failed: DID NOT RAISE <class 'UserWarning'>

    tests/test_types.py:6945: Failed
    ========================= warnings summary =========================
    tests/test_types.py::test_ser_ip_with_unexpected_value
      /tmp/pydantic/pydantic/type_adapter.py:458: UserWarning: Pydantic serializer warnings:
        PydanticSerializationUnexpectedValue(Expected `<class 'ipaddress.IPv4Address'>` but got `<class 'int'>` with value `'123'` - serialized value may not be as expected.)
        return self.serializer.to_python(

    -- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html

The solution is to replace ``pytest.raises()`` with more correct
``pytest.warns()``.  The latter will work correctly, independently
of ``filterwarnings`` value.


.. _custom pytest markers:
   https://docs.pytest.org/en/stable/example/markers.html
.. _pytest-runner: https://pypi.org/project/pytest-runner/
.. _pytest-xdist: https://pypi.org/project/pytest-xdist/
.. _pytest-timeout: https://pypi.org/project/pytest-timeout/
.. _flaky: https://github.com/box/flaky/
.. _pytest-rerunfailures:
   https://github.com/pytest-dev/pytest-rerunfailures/
