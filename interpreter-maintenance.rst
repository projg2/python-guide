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


.. _python repository: https://gitweb.gentoo.org/proj/python.git/
.. _Gentoo fork of CPython repository:
   https://gitweb.gentoo.org/fork/cpython.git/
.. _Docker: https://www.docker.com/
.. _binpkg-docker: https://github.com/mgorny/binpkg-docker
