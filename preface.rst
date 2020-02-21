=======
Preface
=======

Gentoo provides one of the best frameworks for providing Python support
in packages among operating systems.  This includes support for
running multiple versions of Python (while most other distributions
avoid going beyond simultaneous support for Python 2 and one version
of Python 3), alternative implementations of Python, reliable tests,
deep QA checks.  While we aim to keep things simple, this is not always
possible.

At the same time, the available documentation is limited and not always
up-to-date.  Both the `built-in eclass documentation`_ and `Python
project wiki page`_ provide bits of documentation but they are mostly
in reference form and not very suitable for beginners nor people who
do not actively follow the developments within the ecosystem.  This
results in suboptimal ebuilds, improper dependencies, missing tests.

This document aims to fill the gap by providing a good, complete,
by-topic (rather than reference-style) documentation for the ecosystem
in Gentoo and the relevant eclasses.  Combined with examples, it should
help you write good ebuilds and solve common problems as simply
as possible.

`Gentoo Python Guide sources`_ are available on GitHub.  Suggestions
and improvements are welcome.


.. _built-in eclass documentation:
   https://devmanual.gentoo.org/eclass-reference/index.html

.. _Python project wiki page:
   https://wiki.gentoo.org/wiki/Project:Python

.. _Gentoo Python Guide sources:
   https://github.com/mgorny/python-guide/
