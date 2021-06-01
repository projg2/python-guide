============
Porting tips
============

.. highlight:: python

This section highlights some of the known incompatible changes made
in Python that could break Python scripts and modules that used to work
in prior versions.  The sections are split into retroactive changes made
to all Python releases, and information specific to every Python branch
(compared to the previous one).

This guide is by no means considered complete.  If you can think
of other problems you've hit while porting your packages, please let me
know and I will update it.


Retroactive changes
===================

bpo43882_: urlsplit now strips LF, CR and HT characters
-------------------------------------------------------
Changed in: 2.7.18_p9, 3.6.13_p3, 3.7.10_p3, 3.8.9_p2, 3.9.4_p1

Historically, various urllib.parse_ methods have passed special
characters such as LF, CR and HT through into the split URL components.
This could have resulted in various exploits if Python programs did not
validate the resulting components and used them verbatim.

bpo43882_ attempted to address the issue by making urllib.parse_ strip
the three aforementioned characters from the output of its functions.
This fixed one class of potential issues but at the same time opened
another can of worms.  For example, URL validators that used to check
for dangerous special characters in the split URL components stopped
working correctly.  In the best case, the URL were now sanitized instead
of being rejected.  In the worst, the original unparsed URL with
dangerous characters started being passed through.  See e.g. `django
PR#14349`_ for an example of impact and a fix.

Behavior before::

    >>> urllib.parse.urlparse('https://example.com/bad\nurl')
    ParseResult(scheme='https', netloc='example.com', path='/bad\nurl', params='', query='', fragment='')

Behavior after::

    >>> urllib.parse.urlparse('https://example.com/bad\nurl')
    ParseResult(scheme='https', netloc='example.com', path='/badurl', params='', query='', fragment='')


.. _bpo43882: https://bugs.python.org/issue43882
.. _urllib.parse: https://docs.python.org/3/library/urllib.parse.html
.. _django PR#14349: https://github.com/django/django/pull/14349


Python 3.10
===========

See also: `what's new in Python 3.10`_

.. _what's new in Python 3.10:
   https://docs.python.org/3/whatsnew/3.10.html


configure: No package 'python-3.1' found
----------------------------------------
automake prior to 1.16.3 wrongly recognized Python 3.10 as 3.1.
As a result, build with Python 3.10 fails:

.. code-block:: console

    checking for python version... 3.1
    checking for python platform... linux
    checking for python script directory... ${prefix}/lib/python3.10/site-packages
    checking for python extension module directory... ${exec_prefix}/lib/python3.10/site-packages
    checking for PYTHON... no
    configure: error: Package requirements (python-3.1) were not met:

    No package 'python-3.1' found

    Consider adjusting the PKG_CONFIG_PATH environment variable if you
    installed software in a non-standard prefix.

    Alternatively, you may set the environment variables PYTHON_CFLAGS
    and PYTHON_LIBS to avoid the need to call pkg-config.
    See the pkg-config man page for more details.
    Error: Process completed with exit code 1.

To resolve this in ebuild, you need to autoreconf with the Gentoo
distribution of automake::

    inherit autotools

    # ...

    src_prepare() {
        default
        eautoreconf
    }

The upstream fix is to create new distfiles using automake-1.16.3+.


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
