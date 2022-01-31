=============================
Resolving test suite problems
=============================

.. highlight:: bash

Choosing the correct test runner
================================
There are a few modules used to run tests in Python packages.  The most
common include the built-in unittest_ module, pytest_ and nose_.  There
are also some rarely used test tools and domain-specific solutions,
e.g. django_ has its own test runner.  This section will help you
determining which test runner to use and depend on.

Firstly, it is a good idea to look at test sources.  Explicit imports
clearly indicate that a particular test runner needs to be installed,
and most likely used.  For example, if at least one test file has
``import pytest``, pytest is the obvious choice.  If it has ``import
nose``, same goes for nosetests.

In some rare cases the tests may use multiple test packages
simultaneously.  In this case, you need to choose one of the test
runners (see other suggestions) but depend on all of them.

Secondly, some test suites are relying on *implicit* features of a test
runner.  For example, pytest and nose have less strict naming
and structural requirements for test cases.  In some cases, unittest
runner will simply be unable to find all tests.

Thirdly, there are cases when a particular feature of a test runner
is desired even if it is not strictly necessary to run tests.  This
is particularly the case with pytest's output capture that can make
test output much more readable with particularly verbose packages.

Upstream documentation, tox configuration, CI pipelines can provide tips
on the test runner to be used.  However, you should establish whether
this information is wholly correct and up-to-date, and whether
the particular test tool is really desirable.

If the test suite requires no particular runner (i.e. works with
built-in unittest module), using it is preferable to avoid unnecessary
dependencies.  However, you need to make sure that it finds all tests
correctly (i.e. runs no less tests than the alternative) and that it
does not spew too much irrelevant output.

If both pytest and nose seem equally good, the former is recommended
as the latter has ceased development and requires downstream patching.
If you have some free time, convincing upstream to switch from nose
to pytest is a worthwhile goal.


Missing test files in PyPI packages
===================================
One of the more common test-related problems is that PyPI packages
(generated via ``setup.py sdist``) often miss some or all test files.
The latter results in no tests being run, the former in test failures
or errors.

The simplest solution is to use a VCS snapshot instead of the PyPI
tarball::

    # pypi tarballs are missing test data
    #SRC_URI="mirror://pypi/${PN:0:1}/${PN}/${P}.tar.gz"
    SRC_URI="https://github.com/${PN}/${PN}/archive/${PV}.tar.gz -> ${P}.gh.tar.gz"


Checklist for dealing with test failures
========================================
If you see some test failures but do not have a guess as to why they
would be happening, try the following for a start:

1. Check upstream CI (if any).  That's a quick way of verifying that
   there is no known breakage at the relevant tag.

2. Try running tests as your regular user, the way upstream suggests
   (e.g. via ``tox``).  Try using a git checkout at the specific tag.
   This is the basic way of determining whether the package is actually
   broken or if it is something on our end.

3. If the tests fail at the specified tag, try upstream master branch.
   Maybe there was a fix committed.

If it seems that the issue is on our end, try the following and see
if it causes the subset of failing tests to change:

1. Make sure that the test runner is started via ``${EPYTHON}``
   (the eclass-provided ``epytest`` and ``eunittest`` wrappers do that).
   Calling system executables directly (either Python via absolute path
   or system-installed tools that use absolute path in their shebangs)
   may cause just-built modules not to be imported correctly.

2. Try running the test suite from another directory.  If you're seeing
   failures to load compiled extensions, Python may be wrongly importing
   modules from the current directory instead of the build/install tree.
   Some test suite also depend on paths relative to where upstream run
   tests.

3. Switch to PEP 517 mode, add ``distutils_install_for_testing``
   to the test sub-phase or ``--install`` to ``distutils_enable_tests``
   call.  This resolves majority of problems with the test suite
   requiring the package to be installed prior to testing.

4. Actually install the package to the system (with tests disabled).
   This can confirm cases of package for whom the above function
   does not work.  In the worst case, you can set a test self-dependency
   to force users to install the package before testing::

       test? ( ~dev-python/myself-${PV} )

5. Try testing a different Python implementation.  If a subset of tests
   is failing with Python 3.6, see if it still happens with 3.7 or 3.8.
   If 3.8 is passing but 3.9 is not, it's most likely some
   incompatibility upstream did not account for.

6. Run tests with ``FEATURES=-network-sandbox``.  Sometimes lack
   of Internet access causes non-obvious failures.

7. Try a different test runner.  Sometimes the subtle differences
   in how tests are executed can lead to test failures.  But beware:
   some test runners may not run the full set of tests, so verify
   that you have actually fixed them and not just caused them to
   be skipped.


Skipping problematic tests
==========================
While generally it is preferable to fix tests, sometimes you will face
failures that cannot be easily resolved.  This especially applies
to tests that are broken themselves rather than indicating real problems
with the software.  However, in some cases you will even find yourself
ignoring minor test failures.

.. Note::

   When possible, it is preferable to use pytest along with its
   convenient ignore/deselect options to skip problematic tests.
   Using pytest instead of unittest is usually possible.

Tests that are known to fail reliably can be marked as *expected
failures*.  This has the advantage that the test in question will
continue being run and the test suite will report when it unexpectedly
starts passing again.

Expected failures are not supported by the standard Python unittest
module.  It is supported e.g. by pytest.

::

    sed -i -e \
        "/def test_start_params_bug():/i@pytest.mark.xfail(reason='Known to fail on Gentoo')" \
        statsmodels/tsa/tests/test_arima.py || die

Tests that cause inconsistent results, trigger errors, consume
horrendous amounts of disk space or cause another kind of undesirable
mayhem can be *skipped* instead.  Skipping means that they will not be
run at all.

There are multiple ways to skip a test.  You can patch it to use a skip
decorator, possibly with a condition::

    # broken on py2.7, upstream knows
    sed -i -e '5a\
    import sys' \
        -e '/test_null_bytes/i\
    @pytest.mark.skipif(sys.hexversion < 0x03000000, reason="broken on py2")' \
        test/server.py || die

The easy way to skip a test unconditioanlly is to prefix its name with
an underscore::

    # tests requiring specific locales
    sed -i -e 's:test_babel_with_language_:_&:' \
        tests/test_build_latex.py || die
    sed -i -e 's:test_polyglossia_with_language_:_&:' \
        tests/test_build_latex.py || die

Finally, if all tests in a particular file are problematic, you can
simply remove that file.  If all tests belonging to the package
are broken, you can use ``RESTRICT=test`` to disable testing altogether.


Tests requiring Internet access
===============================
One of more common causes of test failures are attempts to use Internet.
With Portage blocking network access by default, packages performing
tests against remote servers often fail.

Ideally, packages would use mocking or replay tests rather than using
real Internet services.  Devmanual provides a detailed explanation `why
tests must not use Internet`_.

Some packages provide explicit methods of disabling network-based tests.
For example, ``dev-python/tox`` provides a switch for that::

    python_test() {
        distutils_install_for_testing
        epytest --no-network
    }

There are packages that skip tests if they fail specifically due to
connection errors, or detect whether Internet is accessible.  Ideally,
you should modify those packages to disable network tests
unconditionally.  For example, ``dev-python/pygit2`` ebuild does this::

    # unconditionally prevent it from using network
    sed -i -e '/def no_network/a \
        return True' test/utils.py || die

In other cases, you will have to explicitly disable these tests.
In some cases, it will be reasonable to remove whole test files or even
restrict tests entirely.

If the package's test suite relies on Internet access entirely and there
is no point in running even a subset of tests, please implement running
tests and combine test restriction with ``PROPERTIES=test_network``
to allow interested users to run tests when possible::

    # users can use ALLOW_TEST=network to override this
    PROPERTIES="test_network"
    RESTRICT="test"

    distutils_enable_tests pytest


Tests aborting (due to assertions)
==================================
There are cases of package's tests terminating with an unclear error
message and backtrace similar to the following::

    ============================= test session starts ==============================
    platform linux -- Python 3.7.8, pytest-6.0.1, py-1.9.0, pluggy-0.13.1 -- /usr/bin/python3.7
    cachedir: .pytest_cache
    rootdir: /var/tmp/portage/dev-python/sabyenc-4.0.2/work/sabyenc-4.0.2, configfile: pytest.ini
    collecting ... collected 24 items

    [...]
    tests/test_decoder.py::test_crc_pickles PASSED                           [ 54%]
    tests/test_decoder.py::test_empty_size_pickles Fatal Python error: Aborted

    Current thread 0x00007f748bc47740 (most recent call first):
      File "/var/tmp/portage/dev-python/sabyenc-4.0.2/work/sabyenc-4.0.2/tests/testsupport.py", line 74 in sabyenc3_wrapper
      File "/var/tmp/portage/dev-python/sabyenc-4.0.2/work/sabyenc-4.0.2/tests/test_decoder.py", line 119 in test_empty_size_pickles
      File "/usr/lib/python3.7/site-packages/_pytest/python.py", line 180 in pytest_pyfunc_call
      File "/usr/lib/python3.7/site-packages/pluggy/callers.py", line 187 in _multicall
      [...]
      File "/usr/lib/python-exec/python3.7/pytest", line 11 in <module>
    /var/tmp/portage/dev-python/sabyenc-4.0.2/temp/environment: line 2934:    66 Aborted                 (core dumped) pytest -vv

This usually indicates that the C code of some Python extension failed
an assertion.  Since pytest does not print captured output when exiting
due to a signal, you need to disable output capture (using ``-s``)
to get a more useful error, e.g.::

    $ python3.7 -m pytest -s
    =============================================================== test session starts ===============================================================
    platform linux -- Python 3.7.8, pytest-6.0.1, py-1.9.0, pluggy-0.13.1
    rootdir: /tmp/sabyenc, configfile: pytest.ini
    plugins: asyncio-0.14.0, forked-1.3.0, xdist-1.34.0, hypothesis-5.23.9, mock-3.2.0, flaky-3.7.0, timeout-1.4.2, freezegun-0.4.2
    collected 25 items                                                                                                                                

    tests/test_decoder.py .............python3.7: src/sabyenc3.c:596: decode_usenet_chunks: Assertion `PyByteArray_Check(PyList_GetItem(Py_input_list, lp))' failed.
    Fatal Python error: Aborted

    Current thread 0x00007fb5db746740 (most recent call first):
      File "/tmp/sabyenc/tests/testsupport.py", line 73 in sabyenc3_wrapper
      File "/tmp/sabyenc/tests/test_decoder.py", line 117 in test_empty_size_pickles
      File "/usr/lib/python3.7/site-packages/_pytest/python.py", line 180 in pytest_pyfunc_call
      File "/usr/lib/python3.7/site-packages/pluggy/callers.py", line 187 in _multicall
      File "/usr/lib/python3.7/site-packages/pluggy/manager.py", line 87 in <lambda>
      [...]
      File "/usr/lib/python3.7/site-packages/pytest/__main__.py", line 7 in <module>
      File "/usr/lib/python3.7/runpy.py", line 85 in _run_code
      File "/usr/lib/python3.7/runpy.py", line 193 in _run_module_as_main
    Aborted (core dumped)

Now the message clearly indicates the failed assertion.

It is also common that upstream is initially unable to reproduce
the bug.  This is because Ubuntu and many other common distributions
build Python with ``-DNDEBUG`` and the flag leaks to extension builds.
As a result, all assertions are stripped at build time.  Upstream
can work around that by explicitly setting ``CFLAGS`` for the build,
e.g.::

    $ CFLAGS='-O0 -g' python setup.py build build_ext -i
    $ pytest -s


Installing extra dependencies in test environment (e.g. example plugins)
========================================================================
Rarely, the test suite expects some package being installed that
does not fit being packaged and installed system-wide.  For example,
isort's tests use a few example plugins that are not useful to end
users, or pip's test suite still requires old virtualenv that collides
with the modern versions.  These problems can be resolved by installing
the packages locally within the ebuild.

.. Warning::

   While the additional packages are installed into a temporary install
   tree, they can leak into the build directory.  Afterwards, they are
   picked by ``setup.py install`` and installed alongside the package.
   If this happens, you need to explicitly remove them
   from ``${BUILD_DIR}/lib`` afterwards.

To do this, just use ``distutils_install_for_testing`` in every package
that you need to install.  For example::

    python_test() {
        # the main package
        distutils_install_for_testing
        # additional plugins
        local p
        for p in example*/; do
            pushd "${p}" >/dev/null || die
            distutils_install_for_testing
            popd >/dev/null || die
        done
        # remove examples leaked into BUILD_DIR
        rm "${BUILD_DIR}"/lib/example* || die

        epytest
    }

If the extra packages are not included in the main distribution tarball,
you will also need to fetch them, e.g.::

    VENV_PV=16.7.10
    SRC_URI+="
        test? (
            https://github.com/pypa/virtualenv/archive/${VENV_PV}.tar.gz
                -> virtualenv-${VENV_PV}.tar.gz
        )
    "

    python_test() {
        distutils_install_for_testing
        pushd "${WORKDIR}/virtualenv-${VENV_PV}" >/dev/null || die
        distutils_install_for_testing
        popd >/dev/null || die
        # prevent it from being installed
        rm -r "${BUILD_DIR}"/lib/virtualenv* || die

        epytest
    }


.. _unittest: https://docs.python.org/3/library/unittest.html
.. _pytest: https://docs.pytest.org/en/latest/
.. _nose: https://github.com/nose-devs/nose
.. _django: https://www.djangoproject.com/
.. _why tests must not use Internet:
   https://devmanual.gentoo.org/ebuild-writing/functions/src_test/#tests-that-require-network-or-service-access
