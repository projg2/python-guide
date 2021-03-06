==============
pytest recipes
==============

.. highlight: bash

Skipping tests based on markers
===============================
A few packages use `custom pytest markers`_ to indicate e.g. tests
requiring Internet access.  These markers can be used to conveniently
disable whole test groups, e.g.::

    python_test() {
        pytest -vv -m 'not network' dask || die "Tests failed with ${EPYTHON}"
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

::

    local deselect=(
        # ignore whole file with missing dep
        --ignore tests/test_client.py
        # deselect a single test
        --deselect 'tests/utils/test_general.py::test_filename'
        # deselect a parametrized test based on first param
        --deselect
        'tests/test_transport.py::test_transport_works[eventlet'
    )
    [[ ${EPYTHON} == python3.6 ]] && deselect+=(
        # deselect a test for py3.6 only
        --deselect
        'tests/utils/test_contextvars.py::test_leaks[greenlet]'
    )
    pytest -vv "${deselect[@]}" || die "Tests failed with ${EPYTHON}"


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


Using pytest-xdist to run tests in parallel
===========================================
pytest-xdist_ is a plugin that makes it possible to run multiple tests
in parallel.  This is especially useful for programs with large test
suites that take significant time to run single-threaded.

Not all test suites support pytest-xdist.  Particularly, it requires
that the tests are written not to collide one with another.

Using pytest-xdist is recommended if the package in question supports it
(i.e. it does not cause semi-random test failures) and its test suite
takes significant time.  When using pytest-xdist, please respect user's
make options for the job number, e.g.::

    inherit multiprocessing

    python_test() {
       pytest -vv \
           -n "$(makeopts_jobs "${MAKEOPTS}" "$(get_nproc)")" ||
           die "Tests failed with ${EPYTHON}"
    }

Please note that some upstream use pytest-xdist even if there is no real
gain from doing so.  If the package's tests take a short time to finish,
please avoid the dependency and strip it if necessary.


Avoiding dependencies on other pytest plugins
=============================================
There is a number of pytest plugins that have little value to Gentoo
users.  They include plugins for test coverage
(``dev-python/pytest-cov``), coding style (``dev-python/pytest-flake8``)
and more.  Generally, packages should avoid using those plugins.

In some cases, upstream packages only list them as dependencies
but do not use them automatically.  In other cases, you will need
to strip options enabling them from ``pytest.ini`` or ``setup.cfg``.

::

    src_prepare() {
        sed -i -e 's:--cov=wheel::' setup.cfg || die
        distutils-r1_src_prepare
    }


Explicitly disabling automatic pytest plugins
=============================================
Besides plugins explicitly used by the package, there are a few pytest
plugins that enable themselves automatically for all test suites
when installed.  In some cases, their presence causes tests of packages
that do not expect them, to fail.

An example of such package used to be ``dev-python/pytest-relaxed``.
To resolve problems due to the plugin, it was necessary to disable
it explicitly::

    python_test() {
        # pytest-relaxed plugin makes our tests fail
        pytest -vv -p no:relaxed || die "Tests fail with ${EPYTHON}"
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
             dev-python/dev-python/pytest-rerunfailures[${PYTHON_USEDEP}]
        )"

    python_test() {
        pytest -vv -p no:flaky || die "Tests failed with ${EPYTHON}"
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
        pytest -vv test || die "Tests failed with ${EPYTHON}"
    }

or::

    python_test() {
        pytest -vv --ignore jaraco || die "Tests failed with ${EPYTHON}"
    }


.. _custom pytest markers:
   https://docs.pytest.org/en/stable/example/markers.html
.. _pytest-runner: https://pypi.org/project/pytest-runner/
.. _pytest-xdist: https://pypi.org/project/pytest-xdist/
.. _flaky: https://github.com/box/flaky/
.. _pytest-rerunfailures:
   https://github.com/pytest-dev/pytest-rerunfailures/
