=============================
Resolving test suite problems
=============================

.. highlight: bash

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


Skipping problematic tests
==========================
While generally it is preferable to fix tests, sometimes you will face
failures that cannot be easily resolved.  This especially applies
to tests that are broken themselves rather than indicating real problems
with the software.  However, in some cases you will even find yourself
ignoring minor test failures.

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
        pytest -vv --no-network || die "Testsuite failed under ${EPYTHON}"
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
restrict tests entirely.  However, even if the package's test suite
relies on Internet access entirely, please implement running tests,
so that an interested user can remove the restriction and run them
if necessary.


.. _why tests must not use Internet:
   https://devmanual.gentoo.org/ebuild-writing/functions/src_test/#tests-that-require-network-or-service-access
