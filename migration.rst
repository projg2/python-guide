================
Migration guides
================


.. index:: PYTHON_MULTI_USEDEP

Migrating from old PYTHON_USEDEP syntax in python-single-r1
===========================================================
Prior to February 2020, ``python-single-r1`` used to provide a single
``PYTHON_USEDEP`` variable alike the two other eclasses.  However,
getting it to work correctly both on single-impl and multi-impl packages
required a gross hack.

The current eclass API requires using ``python_gen_cond_dep`` function
to generate multi-impl deps instead, with ``PYTHON_USEDEP`` serving
as a placeholder.  Single-impl deps can be expressed with
``PYTHON_SINGLE_USEDEP`` that can be used either as placeholder,
or directly as a variable.

During a transitional period, ``PYTHON_USEDEP`` was banned entirely
and ``PYTHON_MULTI_USEDEP`` was used instead.  As of EAPI 8,
the opposite is true — ``PYTHON_USEDEP`` is to be used,
and ``PYTHON_MULTI_USEDEP`` was removed.

The recommended rule of thumb for migrating old ebuilds is to:

1. Replace all instances of ``${PYTHON_USEDEP}`` for simple-impl deps
   with ``${PYTHON_SINGLE_USEDEP}``.  If you don't know the type
   of given dep, dependency check (repoman, pkgcheck) will tell you
   if you chose wrong.

2. Wrap the dependencies using ``${PYTHON_USEDEP}`` in a single
   ``python_gen_cond_dep`` block (reordering may be desirable).

3. Run ``pkgcheck scan`` or ``repoman full``.  If you get syntax errors,
   you probably missed ``python_gen_cond_dep`` or did not escape
   the ``$`` in placeholder properly.  If you get unmatched dependency,
   you probably got single-impl vs. multi-impl wrong.

This way, you can generally convert ebuilds using trial-and-error
method.
