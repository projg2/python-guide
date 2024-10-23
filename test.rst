========================
Tests in Python packages
========================

.. highlight:: bash


Why is running tests important?
===============================
Since Python performs only minimal build-time (or more precisely,
import-time) checking of correctness, it is important to run tests
of Python packages in order to catch any problems early.  This is
especially important for permitting others to verify support for new
Python implementations.


.. index:: distutils_enable_tests

Using distutils_enable_tests
============================

Basic use case
--------------
The simplest way of enabling tests is to call ``distutils_enable_tests``
in global scope, passing the test runner name as the first argument.
This function takes care of declaring test phase, setting appropriate
dependencies and ``test`` USE flag if necessary.  If called after
setting ``RDEPEND``, it also copies it to test dependencies.

.. code-block:: bash
   :emphasize-lines: 26

    # Copyright 1999-2024 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=8

    DISTUTILS_USE_PEP517=setuptools
    PYTHON_COMPAT=( python3_{10..13} pypy3 )

    inherit distutils-r1 pypi

    DESCRIPTION="An easy whitelist-based HTML-sanitizing tool"
    HOMEPAGE="
        https://github.com/mozilla/bleach/
        https://pypi.org/project/bleach/
    "

    LICENSE="Apache-2.0"
    SLOT="0"
    KEYWORDS="~alpha ~amd64 ~arm ~arm64 ~hppa ~ia64 ~mips ~ppc ~ppc64 ~s390 ~sparc ~x86"

    RDEPEND="
        dev-python/packaging[${PYTHON_USEDEP}]
        >=dev-python/html5lib-1.0.1-r1[${PYTHON_USEDEP}]
    "

    distutils_enable_tests pytest

The valid values include:

- ``import-check`` for minimal import checking
  using ``dev-python/pytest-import-check`` (see: `Import-checking
  packages with no working tests`_)
- ``pytest`` for ``dev-python/pytest``
- ``setup.py`` to call ``setup.py test`` (*deprecated*)
- ``unittest`` to use built-in unittest discovery

See `choosing the correct test runner`_ for more information.


Adding more test dependencies
-----------------------------
Additional test dependencies can be specified in ``test?`` conditional.
The flag normally does not need to be explicitly declared,
as ``distutils_enable_tests`` does that in the majority of cases.

Please read the section on `undesirable test dependencies`_ too.

.. code-block:: bash
   :emphasize-lines: 22-24,27

    # Copyright 2023-2024 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=8

    DISTUTILS_USE_PEP517=hatchling
    PYTHON_COMPAT=( python3_{10..13} )

    inherit distutils-r1 pypi

    DESCRIPTION="Reusable constraint types to use with typing.Annotated"
    HOMEPAGE="
        https://github.com/annotated-types/annotated-types/
        https://pypi.org/project/annotated-types/
    "

    LICENSE="MIT"
    SLOT="0"
    KEYWORDS="amd64 arm arm64 ~loong ~mips ppc ppc64 ~riscv ~s390 sparc x86"

    BDEPEND="
        test? (
            dev-python/pytest-mock[${PYTHON_USEDEP}]
        )
    "

    distutils_enable_tests pytest

Note that ``distutils_enable_tests`` modifies variables directly
in the ebuild environment.  This means that if you wish to change their
values, you need to append to them, i.e. the bottom part of that ebuild
can be rewritten as:

.. code-block:: bash
   :emphasize-lines: 3

    distutils_enable_tests pytest

    BDEPEND+="
        test? (
            dev-python/pytest-mock[${PYTHON_USEDEP}]
        )
    "


Installing the package before running tests
-------------------------------------------
In PEP 517 mode, the eclass automatically exposes a venv-style install
tree to the test phase.  No explicit action in necessary.

In the legacy mode, ``distutils_enable_tests`` has an optional
``--install`` option that can be used to force performing an install
to a temporary directory.  More information can be found in the legacy
section.


Customizing the test phase
--------------------------
If additional pre-/post-test phase actions need to be performed,
they can be easily injected via overriding ``src_test()`` and making
it call ``distutils-r1_src_test``:

.. code-block:: bash
   :emphasize-lines: 34-38

    # Copyright 1999-2024 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=8

    DISTUTILS_USE_PEP517=setuptools
    PYTHON_COMPAT=( python3_{10..13} )

    inherit distutils-r1 virtualx pypi

    DESCRIPTION="Extra features for standard library's cmd module"
    HOMEPAGE="
        https://github.com/python-cmd2/cmd2/
        https://pypi.org/project/cmd2/
    "

    LICENSE="MIT"
    SLOT="0"
    KEYWORDS="~amd64 ~arm ~arm64 ~ppc64 ~x86 ~amd64-linux ~x86-linux"

    RDEPEND="
        dev-python/attrs[${PYTHON_USEDEP}]
        >=dev-python/colorama-0.3.7[${PYTHON_USEDEP}]
        >=dev-python/pyperclip-1.6[${PYTHON_USEDEP}]
        dev-python/six[${PYTHON_USEDEP}]
        dev-python/wcwidth[${PYTHON_USEDEP}]
    "
    BDEPEND="
        dev-python/setuptools_scm[${PYTHON_USEDEP}]
    "

    distutils_enable_tests pytest

    src_test() {
        # tests rely on very specific text wrapping...
        local -x COLUMNS=80
        distutils-r1_src_test
    }

If the actual test command needs to be customized, or a non-standard
test tool needs to be used, you can define a ``python_test()`` sub-phase
function.  This function is called for every enabled Python target
by the default ``src_test`` implementation.  This can either be combined
with ``distutils_enable_tests`` call, or used instead of it.  In fact,
the former function simply defines a ``python_test()`` function as part
of its logic.

.. code-block:: bash
   :emphasize-lines: 20,21,25-31,34-36

    # Copyright 1999-2024 Gentoo Authors
    # Distributed under the terms of the GNU General Public License v2

    EAPI=8

    DISTUTILS_USE_PEP517=setuptools
    PYTHON_COMPAT=( python3_{10..13} pypy3 )

    inherit distutils-r1 pypi

    DESCRIPTION="Bash tab completion for argparse"
    HOMEPAGE="
        https://github.com/kislyuk/argcomplete/
        https://pypi.org/project/argcomplete/
    "

    LICENSE="Apache-2.0"
    SLOT="0"
    KEYWORDS="~alpha ~amd64 ~arm ~arm64 ~hppa ~loong ~m68k ~mips ~ppc ~ppc64 ~riscv ~s390 ~sparc ~x86 ~amd64-linux ~x86-linux ~x64-macos"
    IUSE="test"
    RESTRICT="!test? ( test )"

    # pip is called as an external tool
    BDEPEND="
        test? (
            app-shells/fish
            app-shells/tcsh
            app-shells/zsh
            dev-python/pexpect[${PYTHON_USEDEP}]
            >=dev-python/pip-19
        )
    "

    python_test() {
        "${EPYTHON}" test/test.py -v || die
    }

Note that ``python_test`` is called by ``distutils-r1_src_test``,
so you must make sure to call it if you override ``src_test``.


.. index:: epytest

Customizing the test phase for pytest
-------------------------------------
For the relatively frequent case of pytest-based packages needing
additional customization, a ``epytest`` helper is provided.  The helper
runs ``pytest`` with a standard set of options and automatic handling
of test failures.

For example, if upstream uses ``network`` marker to disable
network-based tests, you can override the test phase in the following
way::

    distutils_enable_tests pytest

    python_test() {
        epytest -m 'not network'
    }


.. index:: virtx

Running tests with virtualx
---------------------------
Test suites requiring a display to work correctly can often be appeased
usng Xvfb.  If the package in question does not start Xvfb directly,
``virtualx.eclass`` can be used to do that.  Whenever possible, it is
preferable to run a single server in ``src_test()`` for all passes
of the test suite, e.g.::

    distutils_enable_tests pytest

    src_test() {
        virtx distutils-r1_src_test
    }

Note that ``virtx`` implicitly enables nonfatal mode.  This means that
e.g. ``epytest`` will no longer terminate the ebuild on failure, and you
need to use ``die`` explicitly for it::

    src_test() {
        virtx distutils-r1_src_test
    }

    python_test() {
        epytest -m "not network" || die "Tests failed with ${EPYTHON}"
    }

.. Warning::

   Explicit ``|| die`` is only necessary when overriding ``python_test``
   and running ``epytest`` inside a ``nonfatal``.  The ``virtx`` command
   runs its arguments via a ``nonfatal``.  The default ``python_test``
   implementation created by ``distutils_enable_tests`` accounts for
   this.  In other contexts, ``epytest`` will die on its own.


Choosing the correct test runner
================================
There are a few modules used to run tests in Python packages.  The most
common include the built-in unittest_ module and pytest_.  There
are also some rarely used test tools and domain-specific solutions,
e.g. django_ has its own test runner.  This section will help you
determining which test runner to use and depend on.

Firstly, it is a good idea to look at test sources.  Explicit imports
clearly indicate that a particular test runner needs to be installed,
and most likely used.  For example, if at least one test file has
``import pytest``, pytest is the obvious choice.

In some rare cases the tests may use multiple test packages
simultaneously.  In this case, you need to choose one of the test
runners (see other suggestions) but depend on all of them.

Secondly, some test suites are relying on *implicit* features of a test
runner.  For example, pytest has less strict naming and structural
requirements for test cases.  In some cases, unittest runner will simply
be unable to find all tests.

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


Import-checking packages with no working tests
==============================================
If the package has no tests at all (or the tests are completely
unusable), the ``import-check`` option can be used instead.  This option
uses a dedicated pytest plugin to verify whether all installed Python
packages can be imported.  This includes both Python modules
and compiled extensions, and therefore can e.g. detect undefined
symbols.

Since the function is based on pytest, ``EPYTEST_IGNORE`` can be used
to skip files that are intentionally non-importable.

Note that pytest will also run any tests found in the site-packages
directory.  If this is undesirable, a custom test phase can explicitly
disable the default ``python`` plugin responsible for that, e.g.::

    distutils_enable_tests import-check

    python_test() {
        epytest -p no:python --import-check --pyargs foo
    }


Undesirable test dependencies
=============================
There is a number of packages that are frequently listed as test
dependencies upstream but have little to no value for Gentoo users.
It is recommended to skip those test dependencies whenever possible.
If tests fail to run without them, it is often preferable to strip
the dependencies and/or configuration values enforcing them.

*Coverage testing* establishes how much of the package's code is covered
by the test suite.  While this is useful statistic upstream, it has
no value for Gentoo users who just want to install the package.  This
is often represented by dependencies on ``dev-python/coverage``,
``dev-python/pytest-cov``.  In the latter case, you usually need
to strip ``--cov`` parameter from ``pytest.ini``.

*PEP-8 testing* enforces standard coding style across Python programs.
Issues found by it are relevant to upstream but entirely irrelevant
to Gentoo users.  If the package uses ``dev-python/pep8``,
``dev-python/pycodestyle``, ``dev-python/flake8``, strip that
dependency.

``dev-python/pytest-runner`` is a thin wrapper to run pytest
from ``setup.py``.  Do not use it, just call pytest directly.

``dev-python/tox`` is a convenient wrapper to run tests for multiple
Python versions, in a virtualenv.  The eclass already provides the logic
for the former, and an environment close enough to the latter.  Do not
use tox in ebuilds.


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


ImportErrors for C extensions
=============================
Tests are often invoked in such a way that the Python packages
and modules from the current directory take precedence over these found
in the staging area or build directory.  In fact, this is often
necessary to prevent import collisions — e.g. when modules would
be loaded first from the staging area due to explicit imports
then again from the current directory due to test discovery.

.. Warning::

   Not all packages fail explicitly like that.  In particular,
   if the C extensions are optional, the package may either skip tests
   requiring them or silently fall back to testing the pure Python
   variant.  Special caution is advised when packaging software using
   C extensions with top-level source layout.

Unfortunately, this does not work correctly if C extensions are built
as part of these packages.  Since the package imported relatively
does not include the necessary extensions, the imports fail, e.g.:

.. code-block:: pytb

    ____________________ ERROR collecting systemd/test/test_login.py ____________________
    ImportError while importing test module '/tmp/portage/dev-python/python-systemd-234-r
    2/work/python-systemd-234/systemd/test/test_login.py'.
    Hint: make sure your test modules/packages have valid Python names.
    Traceback:
    /usr/lib/python3.8/site-packages/_pytest/python.py:578: in _importtestmodule
        mod = import_path(self.fspath, mode=importmode)
    /usr/lib/python3.8/site-packages/_pytest/pathlib.py:524: in import_path
        importlib.import_module(module_name)
    /usr/lib/python3.8/importlib/__init__.py:127: in import_module
        return _bootstrap._gcd_import(name[level:], package, level)
    <frozen importlib._bootstrap>:1014: in _gcd_import
        ???
    <frozen importlib._bootstrap>:991: in _find_and_load
        ???
    <frozen importlib._bootstrap>:975: in _find_and_load_unlocked
        ???
    <frozen importlib._bootstrap>:671: in _load_unlocked
        ???
    /usr/lib/python3.8/site-packages/_pytest/assertion/rewrite.py:170: in exec_module
        exec(co, module.__dict__)
    systemd/test/test_login.py:6: in <module>
        from systemd import login
    E   ImportError: cannot import name 'login' from 'systemd' (/tmp/portage/dev-python/python-systemd-234-r2/work/python-systemd-234/systemd/__init__.py)

.. Note::

   Historically, the primary recommendation was to change directory
   prior to running the test suite.  However, this was change since
   the ``rm`` approach is easier in PEP 517 mode, and less likely
   to trigger other issues (e.g. pytest missing its configuration,
   tests relying on relative paths).

The modern preference is to remove the package directory prior to
running the test suite.  In PEP 517 mode, the build system is only run
as part of ``src_compile``, and therefore the original sources are not
needed afterwards.  For example, when the tests are in a separate
directory::

    python_test() {
        rm -rf frozendict || die
        epytest
    }

When the tests are installed as a part of the installed package,
``--pyargs`` option can be used to find them via package lookup::

    python_test() {
        rm -rf numpy || die
        epytest --pyargs numpy
    }

The other possible solution is to change the working directory prior
to running the test suite, either into an arbitrary directory that
does not feature the collision, or into the install directory.
The latter could possible be necessary if the tests are installed
as part of the package, and assume paths relative to the source
directory::

    python_test() {
        cd "${BUILD_DIR}/install$(python_get_sitedir)" || die
        epytest
    }

However, please note that changing the working directory leads to pytest
missing its configuration (``pytest.ini``, ``setup.cfg``,
``pyproject.toml``) which in turn could lead to warnings about missing
marks or misleading test suite problems.


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

3. Either switch to PEP 517 mode (preferred), or
   add ``distutils_install_for_testing`` to the test sub-phase or
   ``--install`` to the ``distutils_enable_tests`` call.
   This resolves the majority of problems that arise from the test suite
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

.. highlight:: console

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


Installing extra dependencies in test environment (PEP 517 mode)
================================================================
Rarely, the test suite expects some package being installed that
does not fit being packaged and installed system-wide.  For example,
isort's tests use a few example plugins that are not useful to end
users, or pip's test suite still requires old virtualenv that collides
with the modern versions.  These problems can be resolved by installing
the packages locally within the ebuild.

The ``distutils-r1.eclass`` provides a ``distutils_pep517_install``
helper that can be used to install additional packages.  Please note
that this helper is intended for expert users only, and special care
needs to be taken when using it.  The function takes a single argument
specifying the destination install root, and installs the package
from the current directory.

For example, ``dev-python/isort`` uses the following test phase
to duplicate the install tree and then install additional packages
into it for the purpose of testing.  Note that ``PATH`` is manipulated
(rather than ``PYTHONPATH``) to use virtualenv-style install root.

.. code-block:: bash

    python_test() {
        cp -a "${BUILD_DIR}"/{install,test} || die
        local -x PATH=${BUILD_DIR}/test/usr/bin:${PATH}

        # Install necessary plugins
        local p
        for p in example*/; do
            pushd "${p}" >/dev/null || die
            distutils_pep517_install "${BUILD_DIR}"/test
            popd >/dev/null || die
        done

        epytest
    }


.. _unittest: https://docs.python.org/3/library/unittest.html
.. _pytest: https://docs.pytest.org/en/latest/
.. _django: https://www.djangoproject.com/
.. _why tests must not use Internet:
   https://devmanual.gentoo.org/ebuild-writing/functions/src_test/#tests-that-require-network-or-service-access
