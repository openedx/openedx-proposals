.. _OEP-18 Python Dependency Management:

OEP-18: Python Dependency Management
####################################

+-----------------+--------------------------------------------------------+
| OEP             | :ref:`OEP-18 <OEP-18 Python Dependency Management>`    |
+-----------------+--------------------------------------------------------+
| Title           | Python Dependencies Management                         |
+-----------------+--------------------------------------------------------+
| Last Modified   | 2026-04-24                                             |
+-----------------+--------------------------------------------------------+
| Authors         | Jeremy Bowman <jbowman@edx.org>                        |
+-----------------+--------------------------------------------------------+
| Arbiter         | Calen Pennington <cale@edx.org>                        |
+-----------------+--------------------------------------------------------+
| Status          | Replaced                                               |
+-----------------+--------------------------------------------------------+
| Type            | Best Practice                                          |
+-----------------+--------------------------------------------------------+
| Created         | 2018-03-27                                             |
+-----------------+--------------------------------------------------------+
| `Review Period` | 2018-03-27 - 2018-04-27                                |
+-----------------+--------------------------------------------------------+
| `Resolution`    | `open-edx-proposals#56 resolution`_                    |
+-----------------+--------------------------------------------------------+

.. _open-edx-proposals#56 resolution: https://github.com/openedx/openedx-proposals/pull/56#pullrequestreview-116976355

.. warning::

   This OEP has been replaced and may be out of date.  For the most up-to-date
   information see :ref:`OEP-67 Standard Tools and Technologies`.

.. contents::
   :local:
   :depth: 2

Abstract
********

Proposes best practices for declaring and maintaining dependencies on other
Python packages in Open edX software repositories.

Motivation
**********

The Open edX project includes dozens of Python software repositories, most of
which depend on certain other Python packages being installed in order to
function correctly.  The simple methods we originally used to do this have
assorted drawbacks that have repeatedly caused problems over the past few
years: accidental upgrades to incompatible versions, strict installation
requirements that restrict the ability of downstream packages to manage their
own dependency versions, lack of clarity regarding the full set of packages
actually depended upon, etc.

Outlined here is a recommended standard for declaring dependencies on other
Python packages which resolves most of these issues and will let us make all
the Open edX Python packages consistent with each other (and many other open
source Python projects) for ease of understanding and maintenance.

Specification
*************

The key to successful Python dependency management in Open edX repositories
is to break it down into five parts:

1. Identify the different contexts in which dependencies will need to be
   installed.
2. For each of these contexts, declare the direct dependencies that will be
   needed.  Use the least restrictive constraints which should allow pip to
   install a working set of dependencies.  We generally no longer want or need
   to manually pin specific versions; ``make upgrade`` will do this for us.
3. Run ``make upgrade`` to generate from the high-level dependencies
   declarations a separate set of requirements files which specify the exact
   package versions that are known to work for a Python `virtualenv`_ created
   for that context.
4. Automate updates of the detailed requirements files for each context.
5. When making deliberate changes to the repository's dependencies, only edit
   the files given to ``*.in`` files in the ``requirements`` directory, never
   the generated ``*.txt`` files.  Both types of files must be committed when
   changed.

.. _virtualenv: https://virtualenv.pypa.io/

Identify Usage Contexts
=======================

The dependencies of Python software are typically installed or run in a
variety of different contexts over the course of developing and using it.
The set of dependencies needed to perform a task can easily vary between these
contexts.  The dependencies for each context will be captured in a
separate file in the ``requirements`` directory.  Here are some common
contexts and the file names often used for them:

* Just the standard set of core dependencies for execution on a production
  server to perform its primary purpose (``base.in``)
* Additional dependencies which are only needed when optional extra features
  of a package are desired (the file is typically named after the
  corresponding key in ``extras_require`` from ``setup.py``)
* Assorted testing libraries to run automated test suites (``test.in``)
* Static code analysis tools to perform code quality checks (``quality.in``)
* The utilities called directly by a CI server, typically including tools such
  as `tox`_ used to create virtualenvs but not the testing, quality, etc.
  libraries that are installed in those virtualenvs (``ci.in``)
* `Sphinx`_ and other utilities used to generate developer documentation
  (``doc.in``)
* Additional utilities needed to perform common development tasks (``dev.in``)
* Utilities that a particular developer likes to use with a repository, but
  aren't strictly needed for any of the regular contexts (``private.in``)

.. _Sphinx: http://www.sphinx-doc.org/

Declare Direct Dependencies
===========================

As indicated above, some of the usage contexts have a standard filename used in
the ``requirements`` directory of an Open edX repository to list dependencies.
Others will have an appropriate filename custom to that repository's unique
context.  Each of these is a ``pip``-compatible `requirements file`_ listing
the direct dependencies needed for that context.  Beyond complying with the
file format, there are a few guidelines each of these files should follow:

* The file should start with a brief comment explaining the context in which
  these dependencies are needed.  Examples can be found in the
  `edx-cookiecutters`_ repository, e.g. under
  ``python-template/{{cookiecutter.placeholder_repo_name}}/requirements/``.
* Each listed dependency should have a brief end-of-line comment explaining
  its primary purpose(s) in this context.  These comments typically start at
  the 37th character, which allows enough room for most package name plus
  constraint specifications while keeping the comments visually aligned.
* Avoid direct links to packages in local directories, GitHub, or other version
  control systems if at all possible; all dependencies should be installed
  from `PyPI`_.  If you think you're in one of the rare circumstances where
  installing a package from a URL is appropriate, see the notes below on
  `Installing Dependencies from URLs`_
* If the dependencies in one context are a superset of those in another one,
  do not repeat the dependencies.  Instead, explicitly include the file
  produced by ``make upgrade`` for the smaller set of dependencies in the
  requirements file for the larger set of dependencies. For example,
  ``test.in`` often includes a line like the following to ensure that the same
  versions of packages used in production for a service will also be used when
  testing it::

  -r base.txt                         # Core dependencies of the service being tested

If the repository contains a ``setup.py`` file defining a Python package, the
base dependencies also need to be specified there.  These can be derived from
``requirements/base.in`` with a Python function declared in
``setup.py`` itself, such as the following:

.. code-block:: python

    def load_requirements(*requirements_paths):
        """
        Load all requirements from the specified requirements files.
        Returns a list of requirement strings.
        """
        requirements = set()
        for path in requirements_paths:
            with open(path) as reqs:
                requirements.update(
                    line.split('#')[0].strip() for line in reqs
                    if is_requirement(line.strip())
                )
        return list(requirements)


    def is_requirement(line):
        """
        Return True if the requirement line is a package requirement;
        that is, it is not blank, a comment, a URL, or an included file.
        """
        return line and not line.startswith(('-r', '#', '-e', 'git+', '-c'))

This can be used to define ``install_requires`` as follows:

.. code-block:: python

    install_requires=load_requirements('requirements/base.in'),

.. _requirements file: https://pip.readthedocs.io/en/1.1/requirements.html
.. _edx-cookiecutters: https://github.com/openedx/edx-cookiecutters
.. _Environment markers: https://www.python.org/dev/peps/pep-0508/#environment-markers
.. _PyPI: https://pypi.org/

Imposing Constraints on Dependencies
====================================

Although we usually want to use the latest available version of our
dependencies in order to take advantage of the latest bug fixes, performance
improvements, and security fixes, we sometimes need to impose some constraints
on the version to be used.  These should be collected in
``requirements/constraints.txt`` so they can be imposed uniformly across all
the repository's requirements files; this is done via a ``-c constraints.txt``
line just under the summary comment of each ``*.in`` file in the
``requirements`` directory.  Some guidelines to keep in mind when populating
this file:

* Version constraints should only be used to exclude dependency versions which
  are known (or strongly suspected) to not work in at least one context.
* Constraints on indirect dependencies (used by dependencies but not directly
  by the code in the repository itself) can be added if needed to enforce a
  compatible version.
* `Environment markers`_ should be used as necessary to indicate dependencies
  which should only be installed on specific operating systems, Python
  versions, etc.
* If a dependency is maintained by edX and only used in a few repositories,
  consider if it should stay pinned to a specific version to facilitate
  managing new releases.  Best practice is to avoid making
  backwards-incompatible new releases whenever possible, but this can require
  excessive effort for a package only used in 1-2 repositories.
* Each constraint should be preceded by a comment explaining why the
  constraint has been imposed.  If there is an issue (either in Jira or an
  upstream issue tracker) for resolving the problem, a link to it should be
  included in the comment.
* Minimum versions should generally not be included here; ``pip-compile``
  always tries to use the latest compatible version in the generated
  requirements files.  If minimum versions need to be specified for use in
  ``setup.py``, those constraints should go in ``requirements/base.in`` as
  explained above.

This file should be periodically reviewed to determine if some of the
constraints are no longer required.

Generate Exact Dependency Specifications
========================================

Although we want to keep our manually edited requirements files very simple,
we need a separate set of requirements files which list every single package
needed for each usage context, with exact versions of each for reproducible
test runs and consistent development and production environments.  We can
generate these automatically using `pip-tools`_, which consists of two related
utilities:

* ``pip-compile`` generates a requirements file from one or more high-level
  input requirements files, listing exact versions of every listed and
  indirect dependency needed to satisfy the given constraints.
* ``pip-sync`` ensures that the current virtualenv contains exactly (and only)
  the packages listed in the given requirements files, installing, upgrading,
  and uninstalling packages as needed.

Open edX packages should use an ``upgrade`` make target to use ``pip-compile``
to automatically update the detailed requirements files
(``requirements/*.txt``) to use the newest available packages which satisfy
the constraints in the direct dependencies files.  These generated files are
then used anywhere that runs a command to install dependencies: ``tox.ini``,
GitHub Actions workflows, the ``requirements`` make target (for updating a local
development environment), etc.

By default ``pip-compile`` uses a cache of calculated dependency relationships
to improve the performance of subsequent runs.  Unfortunately, the results of
this cache are sometimes used even after a new package release has changed the
set of packages it depends on.  To avoid generating incorrect requirements
files due to this, it's best to always use the ``--rebuild`` option for the
first run of ``pip-compile`` during an upgrade.

.. _pip-tools: https://github.com/jazzband/pip-tools

Automate Updates of Exact Dependency Specifications
===================================================

While we want all dependencies explicitly pinned in order to benefit from
consistent testing and development environments, it isn't acceptable to leave
these versions untouched for long stretches of time.  Packages we depend on
routinely release new versions to address security issues, fix bugs, and add
new features.  While we don't necessarily need to update our repositories
every time a new dependency version is released, we do want to keep them
current enough that upgrading a single package to fix a known issue doesn't
require suddenly adapting to a few years' worth of API changes that we didn't
pay attention to.

Each Open edX repository should have the following:

* An ``upgrade`` make target as described above, to update the pinned versions
  of all dependencies (and account for any new or removed indirect
  dependencies).
* An automated test suite with reasonably good code coverage, configured to
  be run on new GitHub pull requests.
* A service configured to periodically auto-generate a GitHub pull request
  that tests the output of running ``make upgrade`` (if it results in any
  changes).  This can either be a service such as `requires.io`_ which tracks
  new releases of Python package dependencies, or a recurring scheduled job.
* At least one designated maintainer who receives notifications of the
  generated pull requests and will merge or fix them as needed.  This
  maintainer should scan the changelog for each upgraded package to look for
  changes that merit closer inspection; services like `requires.io`_
  can make this easier.

.. _requires.io: https://requires.io/

Making Deliberate Changes to Dependencies
=========================================

In addition to the automation described above to keep dependencies current
over time, developers will occasionally need to make deliberate changes to the
set of dependencies.  Common changes include:

* A new dependency is needed to support recent code changes.
* The need for an old dependency was removed.
* A version constraint needs to be added to prevent upgrading to a
  backwards-incompatible release of a required package until appropriate code
  changes can be made.
* The code has been updated to support a newer dependency package version
  which was previously blocked by a version constraint.

Whenever a developer needs to make a deliberate change to the repository's
Python package dependencies, they should do the following:

1. Make the appropriate changes to the ``*.in`` files.
2. Run ``make upgrade`` to regenerate the detailed requirements files.
3. For each package for which the pinned version is changing in the ``*.txt``
   requirements files, look at its changelog to make sure that there
   are no problematic backwards-incompatible changes.  If there are, add
   a version constraint to one of the ``.in`` files to prevent it from being
   upgraded to that release, run ``make upgrade`` again, and file a ticket
   briefly describing the change that needs to be made in order to upgrade
   that package further.  Similarly, if there are new features that the code
   depending on that package should start taking advantage of, file tickets
   explaining what should be done.
4. Check in all of the changed requirements files and wait for the automated
   test results.  If one of the upgrades caused unexpected problems, follow
   the same process as if a backwards-incompatible change had been spotted in
   the changelog (add a version constraint, ``make upgrade``, file a ticket).

Manually editing the ``make upgrade`` output files or only running
``pip-compile`` on a single file should generally be avoided, since it risks
failing to account for changes in indirect dependencies or making the
different requirement files fall out of sync with each other.  And in general,
we would rather err on the side of using newer versions of dependencies than
strictly necessary, rather than avoiding upgrades for fear of breaking things.
If the developer is not confident of their ability to assess whether a change
to the dependencies is appropriate, they should seek assistance from other
developers who are either more experienced or more familiar with that
repository.

Resolving "Could not find a version that matches..."
====================================================

Sometimes ``make upgrade`` or ``pip-compile`` will be unable to find a
suitable version of a dependency for the output file because there are
incompatible version constraints in the input files and/or the stated
installation requirements of the other dependencies.  In cases like this,
add the ``-v`` (or ``--verbose``) flag to ``pip-compile`` to get more
detailed information about which dependencies imposed the conflicting
constraints, so you can decide which package(s) to upgrade or pin to resolve
the issue.  Installing and running `pipdeptree`_ can also sometimes help
identify the problem.

.. _pipdeptree: https://github.com/naiquevin/pipdeptree

Installing Dependencies from URLs
=================================

As noted above, you should almost always avoid installing requirements from a URL
or local directory instead of PyPI.  But there are two circumstances where
it can be appropriate:

* You need to test a release candidate of the dependency to make sure it will
  work with your code.
* You critically need a fix for a package which has not yet been included in
  a release, and you cannot arrange for a release to be made in a timely
  manner.

In all other circumstances, the package should be added to PyPI instead.  If
you do need to include a package at a URL, it must:

* have both the package name and version specified (end with "#egg=NAME==VERSION"), and
* have a comment with a link to a public issue, whose acceptance criteria is to install the dependency from PyPI.

For example::

    # TODO: Install this from PyPI (https://github.com/openedx/blockstore/issues/212).
    git+https://github.com/openedx/blockstore.git@1.2.4#egg=blockstore==1.2.4

Rationale
*********

The practices outlined here help prevent the following problems that we have
encountered in the past:

* A new deployment of an Open edX release fails because an unpinned indirect
  dependency recently released a backwards-incompatible version.
* Tests unrelated to a new code change fail, because an unpinned dependency
  was upgraded to a backwards-incompatible version.  This can be difficult
  to diagnose because the upgrade doesn't appear in the diff of pending
  changes.
* Tests have been running against a particular set of pinned versions for
  years, but we now need to upgrade one (like Django) which requires also
  upgrading several of the other dependencies.  This can force dealing with
  a few years' worth of backwards-incompatible changes in multiple packages
  all at once, whereas dealing with them one at a time every few months in
  smaller pull requests would have been more manageable.
* We have a different version of a dependency installed than we expect,
  because the constraints imposed on pip for choosing a version vary between
  different requirements files and we install them one file at a time.
* We keep using years-old package versions despite the availability of newer
  versions with accumulated bug fixes and performance improvements.
* We install in production environments packages which are only needed for
  testing, because we didn't make a clean distinction between the dependencies
  for different usage contexts.  This slows down deployments.
* We try to exhaustively pin all indirect dependencies manually, but miss some
  (especially when a seemingly innocuous upgrade adds some new dependencies).
* We keep installing a package long after we stopped using it, because nobody
  remembers why it was added to the requirements file (especially true for
  indirect dependencies that were later dropped as requirements of the package
  we use directly).
* We install an exhaustive set of testing dependencies in CI, even though
  we really only need it to run tox and codecov; the rest of the testing
  dependencies are installed in a separate virtualenv created by tox, which
  should have a separate requirements file.
* An attempt to pin dependencies in setup.py (or parse its dependencies
  automatically from a requirements file) forces us to change that package
  before we can upgrade one of those dependencies in another repository
  using that package.
* We add a dependency without realizing that it requires multiple additional
  indirect dependencies; we may have chosen an alternative if that had been
  apparent.

There are several good reasons for the recommendation to avoiding installing
packages from URLs whenever possible:

* Specified VCS branches, commits, and tags can all be deleted from a
  repository at any time, suddenly making it impossible to find and install
  the dependency.
* Editable requirements (starting with "-e ") are downloaded and/or inspected
  with each installation of the requirements file, even if the correct version
  is already installed.  This can significantly slow down updates of installed
  requirements.
* Packages installed from local directories don't reflect any changes to
  package metadata (like required package versions) until the version number
  is incremented or the package is uninstalled; just installing again doesn't
  help.
* Package URLs tend to be long and difficult to read, with the actual name of
  the package hidden in the middle or not even present at all.
* As of this writing, ``pip-tools`` still has a bug in handling packages
  installed from local directories that requires special care to work
  around: `relative local paths are expanded to absolute paths`_.  This can be
  partially worked around via a post-processing script for the generated
  requirements files; an example can be found in `edx-platform`_ at
  ``scripts/post-pip-compile.sh``.
* When installing a package from PyPI, pip will not pull requirements
  from URLs for security reasons (the content of the URLs can
  change). It will only pull requirements from PyPI.
* Dependency URLs are incompatible with automated updates. That is, running
  ``make upgrade`` will not update the version of a dependency specified
  in a URL, even if new, compatible versions are available.

.. _relative local paths are expanded to absolute paths: https://github.com/jazzband/pip-tools/issues/204
.. _edx-platform: https://github.com/openedx/edx-platform

Reference Implementation
************************

Many of the Open edX repositories have already begun to comply with the
recommendations outlined here.  In particular, repositories generated using
`edx-cookiecutters`_ should already be configured correctly.  These may
also be useful for reference:

* `django-user-tasks <https://github.com/openedx/django-user-tasks>`_
* `edx-completion <https://github.com/openedx/completion>`_
* `XQueue <https://github.com/openedx/xqueue/>`_

Rejected Alternatives
*********************

`pipenv`_ is a relatively new utility for managing Python dependencies,
written by Kenneth Reitz (author of the `requests`_ package).  Although it
recently became the default dependency management tool recommendation of the
`Python Packaging User Guide`_, it lacks some features that we strongly want
for Open edX:

* The ability to specify more than 2 sets of dependencies (core and
  development)
* The ability to add comments to the dependencies listing explaining why each
  one is needed
* Indication of which other dependencies caused the inclusion of indirect
  dependencies in the full set of requirements
* Easy interoperability with `tox`_, especially for testing multiple versions
  of a major dependency

As a younger package than ``pip-tools``, it also seems to have more
significant still-unresolved problems, although those are gradually being
fixed.

.. _pipenv: https://docs.pipenv.org/
.. _requests: http://python-requests.org/
.. _Python Packaging User Guide: https://packaging.python.org/tutorials/managing-dependencies/#managing-dependencies
.. _tox: https://tox.readthedocs.io/

Change History
**************

2023-05-19
==========

* Update the standard name for CI requirements files and remove references to
  Travis CI now that we've switched to GitHub Actions
* Add more clarification about what typically goes into a CI requirements file

2023-03-06
==========

* Made language around URL-specified dependencies stricter
* Note that URL-specified dependencies subvert the upgrade process
* Add requirement that URL-specified dependencies be accompanied with
  a comment linking to a follow-up issue
* `Pull request #450 <https://github.com/openedx/openedx-proposals/pull/450>`_

2021-04-30
==========

* Clarify recommendation on using `--rebuild` flag
* Update cookiecutters URL, since the cookie cutter repos were combined
* Use syntax highlighting for code blocks only where appropriate
* Remove reference to deprecated `owner` field from OEP-2
* Remove reference to dead site "allmychanges"
* `Pull request #211 <https://github.com/openedx/openedx-proposals/pull/211>`_

2019-02-20
==========

* Remind that pip ignores transitive URL-based dependencies
* `Pull request #106 <https://github.com/openedx/openedx-proposals/pull/106>`_

2018-10-18
==========

* Add tip about ``pip compile -v``
* `Pull request #82 <https://github.com/openedx/openedx-proposals/pull/82>`_

2018-05-02
==========

* Document created
* `Pull request #56 <https://github.com/openedx/openedx-proposals/pull/56>`_
