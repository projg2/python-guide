================
Migration guides
================


.. index:: PYTHON_USEDEP; python-single-r1

Migrating from PYTHON_USEDEP in python-single-r1
================================================
Prior to February 2020, ``python-single-r1`` used to provide
``PYTHON_USEDEP`` variable alike the two other eclasses.  However,
getting it to work correctly both on single-impl and multi-impl packages
required a gross hack.

The current eclass API requires using ``python_gen_cond_dep`` with
``PYTHON_MULTI_USEDEP`` placeholder for multi-impl deps.  Single-impl
deps can be expressed with ``PYTHON_SINGLE_USEDEP`` that can be used
either as placeholder, or directly as a variable.

The recommended rule of thumb for migrating old ebuilds is to:

1. Replace all instances of ``${PYTHON_USEDEP}`` with
   ``${PYTHON_MULTI_USEDEP}`` for multi-impl deps
   or ``${PYTHON_SINGLE_USEDEP}`` for single-impl deps.  If you don't
   know the type of given dep, you can choose arbitrarily and deptree
   check will tell you if you chose wrong.

2. Wrap the dependencies using ``${PYTHON_MULTI_USEDEP}`` in a single
   ``python_gen_cond_dep`` block (reordering may be desirable).

3. Run ``pkgcheck scan`` or ``repoman full``.  If you get syntax errors,
   you probably missed ``python_gen_cond_dep`` or did not escape
   the ``$`` in placeholder properly.  If you get unmatched dependency,
   you probably got single-impl vs. multi-impl wrong.

This way, you can generally convert ebuilds using trial-and-error
method.
