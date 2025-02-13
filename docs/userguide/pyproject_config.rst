.. _pyproject.toml config:

-----------------------------------------------------
Configuring setuptools using ``pyproject.toml`` files
-----------------------------------------------------

.. note:: New in 61.0.0

.. important::
   If compatibility with legacy builds or versions of tools that don't support
   certain packaging standards (e.g. :pep:`517` or :pep:`660`), a simple ``setup.py``
   script can be added to your project [#setupcfg-caveats]_
   (while keeping the configuration in ``pyproject.toml``):

   .. code-block:: python

       from setuptools import setup

       setup()

Starting with :pep:`621`, the Python community selected ``pyproject.toml`` as
a standard way of specifying *project metadata*.
``Setuptools`` has adopted this standard and will use the information contained
in this file as an input in the build process.

The example below illustrates how to write a ``pyproject.toml`` file that can
be used with ``setuptools``. It contains two TOML tables (identified by the
``[table-header]`` syntax): ``build-system`` and ``project``.
The ``build-system`` table is used to tell the build frontend (e.g.
:pypi:`build` or :pypi:`pip`) to use ``setuptools`` and any other plugins (e.g.
``setuptools-scm``) to build the package.
The ``project`` table contains metadata fields as described by
:doc:`PyPUG:specifications/declaring-project-metadata` guide.

.. _example-pyproject-config:

.. code-block:: toml

   [build-system]
   requires = ["setuptools", "setuptools-scm"]
   build-backend = "setuptools.build_meta"

   [project]
   name = "my_package"
   authors = [
       {name = "Josiah Carberry", email = "josiah_carberry@brown.edu"},
   ]
   description = "My package description"
   readme = "README.rst"
   requires-python = ">=3.7"
   keywords = ["one", "two"]
   license = {text = "BSD-3-Clause"}
   classifiers = [
       "Framework :: Django",
       "Programming Language :: Python :: 3",
   ]
   dependencies = [
       "requests",
       'importlib-metadata; python_version<"3.8"',
   ]
   dynamic = ["version"]

   [project.optional-dependencies]
   pdf = ["ReportLab>=1.2", "RXP"]
   rest = ["docutils>=0.3", "pack ==1.1, ==1.3"]

   [project.scripts]
   my-script = "my_package.module:function"

   # ... other project metadata fields as specified in:
   #     https://packaging.python.org/en/latest/specifications/declaring-project-metadata/

.. _setuptools-table:

Setuptools-specific configuration
=================================

.. warning::
   Support for declaring configurations not standardized by :pep:`621`
   (i.e.  the ``[tool.setuptools]`` table),
   is still in **beta** stage and might change in future releases.

While the standard ``project`` table in the ``pyproject.toml`` file covers most
of the metadata used during the packaging process, there are still some
``setuptools``-specific configurations that can be set by users that require
customization.
These configurations are completely optional and probably can be skipped when
creating simple packages.
They are equivalent to the :doc:`/references/keywords` used by the ``setup.py``
file, and can be set via the ``tool.setuptools`` table:

========================= =========================== =========================
Key                       Value Type (TOML)           Notes
========================= =========================== =========================
``platforms``             array
``zip-safe``              boolean                     If not specified, ``setuptools`` will try to guess
                                                      a reasonable default for the package
``eager-resources``       array
``py-modules``            array                       See tip below
``packages``              array or ``find`` directive See tip below
``package-dir``           table/inline-table          Used when explicitly listing ``packages``
``namespace-packages``    array                       **Deprecated** - Use implicit namespaces instead (:pep:`420`)
``package-data``          table/inline-table          See :doc:`/userguide/datafiles`
``include-package-data``  boolean                     ``True`` by default
``exclude-package-data``  table/inline-table
``license-files``         array of glob patterns      **Provisional** - likely to change with :pep:`639`
                                                      (by default: ``['LICEN[CS]E*', 'COPYING*', 'NOTICE*', 'AUTHORS*']``)
``data-files``            table/inline-table          **Discouraged** - check :doc:`/userguide/datafiles`
``script-files``          array                       **Deprecated** - equivalent to the ``script`` keyword in ``setup.py``
                                                      (should be avoided in favour of ``project.scripts``)
``provides``              array                       **Ignored by pip**
``obsoletes``             array                       **Ignored by pip**
========================= =========================== =========================

.. note::
   The `TOML value types`_ ``array`` and ``table/inline-table`` are roughly
   equivalent to the Python's :obj:`list` and :obj:`dict` data types, respectively.

Please note that some of these configurations are deprecated or at least
discouraged, but they are made available to ensure portability.
New packages should avoid relying on deprecated/discouraged fields, and
existing packages should consider alternatives.

.. tip::
   When both ``py-modules`` and ``packages`` are left unspecified,
   ``setuptools`` will attempt to perform :ref:`auto-discovery`, which should
   cover most popular project directory organization techniques, such as the
   :ref:`src-layout` and the :ref:`flat-layout`.

   However if your project does not follow these conventional layouts
   (e.g. you want to use a ``flat-layout`` but at the same time have custom
   directories at the root of your project), you might need to use the ``find``
   directive [#directives]_ as shown below:

   .. code-block:: toml

      [tool.setuptools.packages.find]
      where = ["src"]  # list of folders that contain the packages (["."] by default)
      include = ["my_package*"]  # package names should match these glob patterns (["*"] by default)
      exclude = ["my_package.tests*"]  # exclude packages matching these glob patterns (empty by default)
      namespaces = false  # to disable scanning PEP 420 namespaces (true by default)

   Note that the glob patterns in the example above need to be matched
   by the **entire** package name. This means that if you specify ``exclude = ["tests"]``,
   modules like ``tests.my_package.test1`` will still be included in the distribution
   (to remove them, add a wildcard to the end of the pattern: ``"tests*"``).

   Alternatively, you can explicitly list the packages in modules:

   .. code-block:: toml

      [tool.setuptools]
      packages = ["my_package"]


.. _dynamic-pyproject-config:

Dynamic Metadata
================

Note that in the first example of this page we use ``dynamic`` to identify
which metadata fields are dynamically computed during the build by either
``setuptools`` itself or the plugins installed via ``build-system.requires``
(e.g. ``setuptools-scm`` is capable of deriving the current project version
directly from the ``git`` :wiki:`version control` system).

Currently the following fields can be listed as dynamic: ``version``,
``classifiers``, ``description``, ``entry-points``, ``scripts``,
``gui-scripts`` and ``readme``.
When these fields are expected to be provided by ``setuptools`` a
corresponding entry is required in the ``tool.setuptools.dynamic`` table
[#entry-points]_. For example:

.. code-block:: toml

   # ...
   [project]
   name = "my_package"
   dynamic = ["version", "readme"]
   # ...
   [tool.setuptools.dynamic]
   version = {attr = "my_package.VERSION"}
   readme = {file = ["README.rst", "USAGE.rst"]}

In the ``dynamic`` table, the ``attr`` directive [#directives]_ will read an
attribute from the given module [#attr]_, while ``file`` will read the contents
of all given files and concatenate them in a single string.

========================== =================== =================================================================================================
Key                        Directive           Notes
========================== =================== =================================================================================================
``version``                ``attr``, ``file``
``readme``                 ``file``            Here you can also set ``"content-type"``:

                                               ``readme = {file = ["README", "USAGE"], content-type = "text/plain"}``

                                               If ``content-type`` is not given, ``"text/x-rst"`` is used by default.
``description``            ``file``            One-line text (no line breaks)
``classifiers``            ``file``            Multi-line text with one classifier per line
``entry-points``           ``file``            INI format following :doc:`PyPUG:specifications/entry-points`
                                               (``console_scripts`` and ``gui_scripts`` can be included)
``dependencies``           ``file``            *subset* of the ``requirements.txt`` format
                                               (``#`` comments and blank lines excluded) **BETA**
``optional-dependencies``  ``file``            *subset* of the ``requirements.txt`` format per group
                                               (``#`` comments and blank lines excluded) **BETA**
========================== =================== =================================================================================================

Supporting ``file`` for dependencies is meant for a convenience for packaging
applications with possibly strictly versioned dependencies.

Library packagers are discouraged from using overly strict (or "locked")
dependency versions in their ``dependencies`` and ``optional-dependencies``.

Currently, when specifying ``optional-dependencies`` dynamically, all of the groups
must be specified dynamically; one can not specify some of them statically and
some of them dynamically.

Also note that the file format for specifying dependencies resembles a ``requirements.txt`` file,
however please keep in mind that all non-comment lines must conform with :pep:`508`
(``pip``-specify syntaxes, e.g. ``-c/-r/-e`` flags, are not supported).


.. attention::
   When using the ``file`` directive, please make sure that all necessary
   files are included in the ``sdist``. You can do that via ``MANIFEST.in``
   or using plugins such as ``setuptools-scm``.
   Please have a look on :doc:`/userguide/miscellaneous` for more information.

----

.. rubric:: Notes

.. [#setupcfg-caveats] ``pip`` may allow editable install only with ``pyproject.toml``
   and ``setup.cfg``. However, this behavior may not be consistent over various ``pip``
   versions and other packaging-related tools
   (``setup.py`` is more reliable on those scenarios).

.. [#entry-points] Dynamic ``scripts`` and ``gui-scripts`` are a special case.
   When resolving these metadata keys, ``setuptools`` will look for
   ``tool.setuptool.dynamic.entry-points``, and use the values of the
   ``console_scripts`` and ``gui_scripts`` :doc:`entry-point groups
   <PyPUG:specifications/entry-points>`.

.. [#directives] In the context of this document, *directives* are special TOML
   values that are interpreted differently by ``setuptools`` (usually triggering an
   associated function). Most of the times they correspond to a special TOML table
   (or inline-table) with a single top-level key.
   For example, you can have the ``{find = {where = ["src"], exclude=["tests*"]}}``
   directive for ``tool.setuptools.packages``, or ``{attr = "mymodule.attr"}``
   directive for ``tool.setuptools.dynamic.version``.

.. [#attr] ``attr`` is meant to be used when the module attribute is statically
   specified (e.g. as a string, list or tuple). As a rule of thumb, the
   attribute should be able to be parsed with :func:`ast.literal_eval`, and
   should not be modified or re-assigned.

.. _TOML value types: https://toml.io/en/v1.0.0
