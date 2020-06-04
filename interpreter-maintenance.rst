=====================================
Maintenance of Python implementations
=====================================

Life cycle of a Python implementation
=====================================
Every Python implementation (understood as a potential target) in Gentoo
follows roughly the following life cycle:

1. The interpreter is added to ``~arch`` for initial testing.  At this
   point, packages can not declare support for the implementation yet.

2. The new Python target is added.  It is initially stable-masked,
   so only ``~arch`` users can use it.  At this point, packages start
   being tested against the new target and its support starts being
   declared in ``PYTHON_COMPAT``.

3. When ready, the new interpreter is stabilized.  The target is not yet
   available for stable users, though.

4. The stable-mask for the target is removed.  For this to happen,
   the inconsistencies in stable graph need to be addressed first
   via stabilizing newer versions of packages.

5. Over time, developers are repeatedly asked to push testing packages
   for the new target forward and stabilize new versions supporting it.
   Eventually, the final push for updates happens and packages
   not supporting the new target start being removed.

6. If applicable, the new target becomes the default.  The developers
   are required to test new packages against it.  The support for old
   target is slowly being discontinued.

7. Eventually, the target becomes replaced by the next one.  When it
   nears end of life, the final packages requiring it are masked for
   removal and the target flags are disabled.

8. The compatibility declarations are cleaned up from ``PYTHON_COMPAT``
   and obsolete ebuild and eclass code is cleaned up.

9. Finally, the interpreter is moved to `python repository`_ where it
   lives for as long as it builds.

For example, Python 3.9 is at stage 1 at the time of writing.  It is
still in alpha stage, and upstream has not finalized its feature set,
therefore it is too early to declare package support for Python 3.9
and there are no target flags.

Python 3.8 is moving from stage 2 to stage 3 — it is being stabilized
by arch teams at this very moment.  When that's done, we will work
on unmasking the flag on stable systems and it will become our next
default target.

Python 3.7 is moving from stage 5 to stage 6.  The vast majority
of packages have been ported to it, and we have already announced
the switch date.

When the switch happens, Python 3.6 will move from stage 6 to stage 7.
We are going to support it for quite some time still but as things
progress, we will eventually decide to remove it.

Python 3.5 and 3.4 are at stage 9.  They live in the Python repository
but have no targets.  You can still use them e.g. inside a virtualenv
to test your own software.

Python 2.7 is currently somewhere between stages 6 and 7.  It is still
enabled by default for backwards compatibility but we are aggressively
removing it.

PyPy3 has recently reached stage 3.  It is not clear if we are going
to pursue enabling the target on stable system though.  PyPy2.7 is
at stage 8, as the targets were removed already and it is kept
as a dependency and testing target.


Notes specific to Python interpreters
=====================================
CPython patchsets
-----------------
Gentoo is maintaining patchsets for all CPython versions.  These include
some non-upstreamable Gentoo patches and upstream backports.  While it
is considered acceptable to add a new patch (e.g. a security bug fix)
to ``files/`` directory, it should be eventually moved into
the respective patchset.

When adding a new version, it is fine to use an old patchset if it
applies cleanly.  If it does not, you should regenerate the patchset
for new version.

The origin for Gentoo patches are the ``gentoo-*`` tags the `Gentoo fork
of CPython repository`_.  The recommended workflow is to clone
the upstream repository, then add Gentoo fork as a remote, e.g.::

    git clone https://github.com/python/cpython
    cd cpython
    git remote add gentoo git@git.gentoo.org:fork/cpython.git
    git fetch --tags gentoo

In order to rebase the patchset, check out the tag corresponding
to the previous patchset version and rebase it against the upstream
release tag::

    git checkout gentoo-3.7.4
    git rebase v3.7.6

You may also add additional changes via ``git cherry-pick``.  Once
the new patches are ready, create the tarball and upload it, then
create the tag and push it::

    mkdir python-gentoo-patches-3.7.6
    cd python-gentoo-patches-3.7.6
    git format-patch v3.7.6
    cd ..
    tar -cf python-gentoo-patches-3.7.6.tar python-gentoo-patches-3.7.6
    xz -9 python-gentoo-patches-3.7.6.tar
    scp python-gentoo-patches-3.7.6.tar.xz ...
    git tag gentoo-3.7.6
    git push --tags gentoo


PyPy
----
Due to high resource requirements and long build time, PyPy on Gentoo
is provided both in source and precompiled form.  This creates a bit
unusual ebuild structure:

- ``dev-python/pypy-exe`` provides the PyPy executable and generated
  files built from source,
- ``dev-python/pypy-exe-bin`` does the same in precompiled binary form,
- ``dev-python/pypy`` combines the above with the common files.  This
  is the package that runs tests and satisfies the PyPy target.

Matching ``dev-python/pypy3*`` exist for PyPy3.

When bumping PyPy, ``pypy-exe`` needs to be updated first.  Then it
should be used to build a binary package and bump ``pypy-exe-bin``.
Technically, ``pypy`` can be bumped after ``pypy-exe`` and used to test
it but it should not be pushed before ``pypy-exe-bin`` is ready, as it
would force all users to switch to source form implicitly.

The binary packages are built using Docker_ nowadays, using
binpkg-docker_ scripts.  To produce them, create a ``local.diff``
containing changes related to PyPy bump and run ``amd64-pypy``
(and/or ``amd64-pypy3``) and ``x86-pypy`` (and/or ``x86-pypy3``) make
targets::

    git clone https://github.com/mgorny/binpkg-docker
    cd binpkg-docker
    (cd ~/git/gentoo && git diff origin) > local.diff
    make amd64-pypy amd64-pypy3 x86-pypy x86-pypy3

The resulting binary packages will be placed in your home directory,
in ``~/binpkg/${arch}/pypy``.  Upload them and use them to bump
``pypy-exe-bin``.


Adding a new Python implementation
==================================
Eclass and profile changes
--------------------------
When adding a new Python target, please remember to perform all
the following tasks:

- add the new target flags to ``profiles/desc/python_targets.desc``
  and ``python_single_target.desc``.

- force the new implementation on ``dev-lang/python-exec``
  via ``profiles/base/package.use.force``.

- mask the new target flags on stable profiles
  via ``profiles/base/use.stable.mask``.

- add the new target to ``_PYTHON_ALL_IMPLS`` and update the patterns
  in ``_python_impl_supported()`` in ``python-utils-r1.eclass``.

- add the new implementation to the list
  in ``app-portage/gpyutils/files/implementations.txt``.


Porting initial packages
------------------------
The initial porting is quite hard due to a number of circular
dependencies.  To ease the process, some of the high profile packages
are ported first with tests and their dependencies disabled for the new
implementation, e.g.:

.. code-block:: bash
   :emphasize-lines: 4-11,19-22

    BDEPEND="
        app-arch/unzip
        test? (
            $(python_gen_cond_dep '
                dev-python/mock[${PYTHON_USEDEP}]
                dev-python/pip[${PYTHON_USEDEP}]
                >=dev-python/pytest-3.7.0[${PYTHON_USEDEP}]
                dev-python/pytest-fixture-config[${PYTHON_USEDEP}]
                dev-python/pytest-virtualenv[${PYTHON_USEDEP}]
                dev-python/wheel[${PYTHON_USEDEP}]
            ' python2_7 python3_{6,7,8} pypy3)
            $(python_gen_cond_dep '
                dev-python/futures[${PYTHON_USEDEP}]
            ' -2)
        )
    "

    python_test() {
        if [[ ${EPYTHON} == python3.9 ]]; then
            einfo "Tests are skipped on py3.9 due to unported deps"
            return
        fi

        distutils_install_for_testing
        # test_easy_install raises a SandboxViolation due to ${HOME}/.pydistutils.cfg
        # It tries to sandbox the test in a tempdir
        HOME="${PWD}" pytest -vv ${PN} || die "Tests failed under ${EPYTHON}"
    }


The recommended process is to, in order:

1. Port ``dev-python/setuptools`` and ``dev-python/certifi`` with tests
   disabled.  Test it via ``tox`` in a git checkout.

2. Port ``dev-python/nose`` with additional dependencies disabled
   (tests skip missing dependencies gracefully).

3. Port ``dev-python/pytest`` and its runtime dependencies with pytest's
   tests disabled (but tests of the dependencies enabled).  This should
   yield around 20 packages.  Test it via ``tox`` in a git checkout.

4. Port ``dev-python/urllib3`` and its runtime dependencies with
   urllib3's tests disabled (but tests of the dependencies enabled).
   This should yield another 20 packages.  Test it from a git checkout
   (it uses nox, so you may want to write ``tox.ini`` yourself).

Once these packages are done, you should be able to work towards
reenabling tests in them via porting their (deep) dependencies in groups
of around 10 packages without cyclic dependencies extending out
of the group.


.. _python repository: https://gitweb.gentoo.org/proj/python.git/
.. _Gentoo fork of CPython repository:
   https://gitweb.gentoo.org/fork/cpython.git/
.. _Docker: https://www.docker.com/
.. _binpkg-docker: https://github.com/mgorny/binpkg-docker
