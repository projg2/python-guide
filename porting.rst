============
Porting tips
============

This section highlights some of the known incompatible changes made
in Python that could break Python scripts and modules that used to work
in prior versions.  The sections are split into retroactive changes made
to all Python releases, and information specific to every Python branch
(compared to the previous one).

This guide is by no means considered complete.  If you can think
of other problems you've hit while porting your packages, please let me
know and I will update it.


Python 3.8
==========

See also: `what's new in Python 3.8`_

.. _what's new in Python 3.8:
   https://docs.python.org/3/whatsnew/3.8.html


python-config and pkg-config no longer list Python library by default
---------------------------------------------------------------------
Until Python 3.7, the ``python-X.Y`` pkg-config file and python-config
tool listed the Python library.  Starting with 3.8, this is no longer
the case.  If you are building Python extensions, this is fine (they
are not supposed to link directly to libpython).

If you are building programs that need to embed the Python interpreter,
new ``python-X.Y-embed`` pkg-config file and ``--embed`` parameter
are provided for the purpose.

.. code-block:: console

    $ pkg-config --libs python-3.7
    -lpython3.7m
    $ pkg-config --libs python-3.8

    $ pkg-config --libs python-3.8-embed
    -lpython3.8

To achieve backwards compatibility, you should query
``python-X.Y-embed`` first and fall back to ``python-X.Y``.
