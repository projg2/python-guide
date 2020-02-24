======================
Expert python-r1 usage
======================

.. highlight:: bash

The APIs described in this chapter are powerful but even harder to use
than those described in ``python-r1`` chapter.  You should not consider
using them unless you have a proper ninja training.


.. index:: python_gen_any_dep; python-r1
.. index:: python_check_deps; python-r1

Disjoint build dependencies (any-r1 API)
========================================
Some packages have disjoint sets of runtime and pure build-time
dependencies.  The former need to be built for all enabled
implementations, the latter only for one of them.  The any-r1 API
in ``python-r1`` is specifically suited for expressing that.

Let's consider an example package that uses Sphinx with a plugin
to build documentation.  Naturally, you're going to build the documents
only once, not separately for every enabled target.


Using regular python-r1 API
---------------------------
If you were using the regular API, you'd have to use
``${PYTHON_USEDEP}`` on the dependencies.  The resulting code could look
like the following::

    BDEPEND="
        doc? (
            dev-python/sphinx[${PYTHON_USEDEP}]
            dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]
        )"

    src_compile() {
        ...

        if use doc; then
            python_setup
            emake -C docs html
        fi
    }

If your package is built with support for Python 3.6, 3.7 and 3.8,
then this dependency string will enforce the same targets for Sphinx
and the theme.  However, in practice it will only be used through
Python 3.8.  Normally, this is not such a big deal.

Now imagine your package supports Python 2.7 as well, while Sphinx
does not anymore.  This means that your package will force downgrade
to the old version of ``dev-python/sphinx`` even though it will not
be used via Python 2.7 at all.


Using any-r1 API with python-r1
-------------------------------
As the name suggests, the any-r1 API resembles the API used
by ``python-any-r1`` eclass.  The disjoint build-time dependencies
are declared using ``python_gen_any_dep``, and need to be tested
via ``python_check_deps()`` function.  The presence of the latter
function activates the alternate behavior of ``python_setup``.  Instead
of selecting one of the enabled targets, it will run it to verify
installed dependencies and use one having all dependencies satisfied.

.. code-block:: bash
   :emphasize-lines: 3-6,9-12,18

    BDEPEND="
        doc? (
            $(python_gen_any_dep '
                dev-python/sphinx[${PYTHON_USEDEP}]
                dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]
            ')
        )"

    python_check_deps() {
        has_version "dev-python/sphinx[${PYTHON_USEDEP}]" &&
        has_version "dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]"
    }

    src_compile() {
        ...

        if use doc; then
            python_setup
            emake -C docs html
        fi
    }

Note that ``python_setup`` may select an implementation that is not even
enabled via ``PYTHON_TARGETS``.  The goal is to try hard to avoid
requiring user to change USE flags on dependencies if possible.

An interesting side effect of that is that the supported targets
in the dependencies can be a subset of the one in package.  For example,
we have used this API to add Python 3.8 support to packages before
``dev-python/sphinx`` supported it — the eclass implicitly forced using
another implementation for Sphinx.


Different sets of build-time dependencies
-----------------------------------------
Let's consider the case when Python is used at build-time for something
else still.  In that case, we want ``python_setup`` to work
unconditionally but enforce dependencies only with ``doc`` flag enabled.

.. code-block:: bash
   :emphasize-lines: 9-13,16

    BDEPEND="
        doc? (
            $(python_gen_any_dep '
                dev-python/sphinx[${PYTHON_USEDEP}]
                dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]
            ')
        )"

    python_check_deps() {
        use doc || return 0
        has_version "dev-python/sphinx[${PYTHON_USEDEP}]" &&
        has_version "dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]"
    }

    src_compile() {
        python_setup

        ...

        use doc && emake -C docs html
    }

Note that ``python_setup`` behaves according to the any-r1 API here.
While it will not enforce doc dependencies with ``doc`` flag disabled,
it will use *any* interpreter that is supported and installed, even
if it is not enabled explicitly in ``PYTHON_TARGETS``.


Using any-r1 API with distutils-r1
----------------------------------
The alternate build dependency API also integrates with ``distutils-r1``
eclass.  If ``python_check_deps()`` is declared, the ``python_*_all()``
sub-phase functions are called with the interpreter selected according
to any-r1 rules.

.. code-block:: bash
   :emphasize-lines: 3-6,9-13

    BDEPEND="
        doc? (
            $(python_gen_any_dep '
                dev-python/sphinx[${PYTHON_USEDEP}]
                dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]
            ')
        )"

    python_check_deps() {
        use doc || return 0
        has_version "dev-python/sphinx[${PYTHON_USEDEP}]" &&
        has_version "dev-python/sphinx_rtd_theme[${PYTHON_USEDEP}]"
    }

    python_compile_all() {
        use doc && emake -C docs html
    }

Note that ``distutils-r1`` calls ``python_setup`` unconditionally,
therefore ``python_check_deps()`` needs to account for that.

Normally you won't have to use this API for Sphinx though —
``distutils_enable_sphinx`` does precisely that for you.
