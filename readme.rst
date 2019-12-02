########################################################################
Documentation repository for `oracle.rtfd.io <http://oracle.rtfd.io>`_
########################################################################

Oracle SQL & PL/SQL optimization for developers is a `Sphinx <http://sphinx-doc.org>`_ documentation project initiated by `Ian Hellström <https://databaseline.tech>`_.
Its aim is to aid developers in writing efficient SQL & PL/SQL code for modern Oracle databases.
It is also a chronicle of successful and not so successful experiments as a developer for various production systems.

This public Git repository hosts the reStructuredText (reST) files needed to (automatically) build the documentation hosted on `RTD <http://readthedocs.org>`_.

The `default adornments <http://sphinx-doc.org/rest.html#sections>`_ for section headers are used:

* ``#`` (with overline) for parts
* ``*`` (with overline) for chapters
* ``=`` for sections
* ``-`` for subsections
* ``^`` for subsubsections
* ``"`` for paragraphs

Since the interpretation of these adornments is relative to each file in which they are used, it is recommended that all chapters belonging to a part, all sections belonging to a chapter, and so on, are, if defined in separate files, combined with the ``include`` directive at the highest level, i.e. (part, chapter, ...).
The ``include`` directive reads the contents of each file in the context of the document in which it is written.
Please check the source folder of the repository for examples.

****************
Acknowledgements
****************
Many thanks are due to the team at `Read The Docs <http://readthedocs.org>`_ for allowing the automated build and distribution of this document.
For hosting the documentation files we are also indebted to `Atlassian <http://bitbucket.org>`_.

*************
Contributions
*************
Interested in contributing to the project?
Great!
Drop us a `line <https://databaseline.tech>`_ any time.
