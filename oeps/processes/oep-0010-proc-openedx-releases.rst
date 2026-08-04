.. _OEP-10 Open edX Releases:

OEP-10: Open edX Releases
#########################

+---------------+---------------------------------------------------+
| OEP           | :ref:`OEP-10 <OEP-10 Open edX Releases>`          |
+---------------+---------------------------------------------------+
| Title         | Open edX Releases                                 |
+---------------+---------------------------------------------------+
| Last Modified | 2020-04-26                                        |
+---------------+---------------------------------------------------+
| Author        | Ned Batchelder <ned@edx.org>                      |
+---------------+---------------------------------------------------+
| Arbiter       | Jeremy Bowman                                     |
+---------------+---------------------------------------------------+
| Status        | Accepted                                          |
+---------------+---------------------------------------------------+
| Type          | Process                                           |
+---------------+---------------------------------------------------+
| Created       | 2016-10-14                                        |
+---------------+---------------------------------------------------+
| Resolution    | `Original pull request`_                          |
+---------------+---------------------------------------------------+

.. _Original pull request: https://github.com/openedx/openedx-proposals/pull/26

..
    - Expectations for component owners


Abstract
********

Open edX is packaged occasionally into releases. This document details the
process.


Motivation
**********

Open edX releases happen roughly every six months.  This document standardizes
aspects of the release process to ensure that all involved understand and
participate appropriately.


Specification
*************


Components
==========

When talking about software that is part of Open edX, there are a number of
components that might be useful to discuss, of various sizes:

- Application: this is a user-visible application that provides some useful
  functionality.  Examples are the LMS, Studio, Analytics Insights.  These can
  be composed of a number of different repos and services.

- Service (or IDA): this is a program that can be separately installed and
  started.  Often applications will need a number of these working together to
  provide all functionality.  The LMS and Studio applications are implemented
  by the edxapp service, which also uses the forums service.  A service might
  or might not have a user interface.

- Feature: large applications have major features which are configurable, and
  should be called out explicitly if their support status is different than the
  application they are a part of.

- Library: code that is named as a dependency of another application, service,
  or library.


Levels of support
=================

Components in Open edX are either *included* or *supported* in a release.

To be **included** in an Open edX release, a component must meet these
criteria.

- It must be successfully merged into its repository's release-from branch.
  This is typically master, though it can be any branch.  For a component that
  a release pins by version rather than by branch (see `Release creation`_
  below), this means the pinned version must have been published from a branch
  that is maintained for the life of the release.

- It must be installable in at least one of the supported installation methods.

- It must be usable in the installed configuration.

Beyond inclusion, to be **supported** in an Open edX release, a component must
meet further criteria:

- It must be documented thoroughly and accurately.

- A maintainer (or maintainer group) must agree to be
  minimally available for answering questions in the public channels.

- A maintainer (or maintainer group) must agree to provide
  fixes if severe problems are identified.

Many of these criteria are open to interpretation, or varying degrees of
effort. For example, what does "documented thoroughly" mean? We can't quantify
that. It and other loose criteria are included here because they are important
parts of providing a finished quality product, and we don't want to overlook
them.


Dependencies
============

Our software is built atop other software layers supported by their creators.
It is important to consider the support windows for those layers when choosing
which version to use.  An Open edX release is supported by the community until the next
release, so for about six months.  The supporting layers must be supported by
their developers for the entire Open edX release.

Typically this means choosing Long Term Support (LTS) versions of the
supporting layers, but it's possible shorter-term support versions will provide
the support needed.

The layers in question here are Django, Python, and Ubuntu.  Here's a `calendar
of the known support windows`__ and how they overlap with Open edX plans.

.. __: https://docs.google.com/spreadsheets/d/1wtpoypH1XOPc_G6h9AUNXJ6XiNKD6dlkMP3lubdpE9I


Release creation
================

A new release line is created roughly every six months.  Release lines are
named with words in alphabetical order: Dogwood, Eucalyptus, Ficus, Gingko,
and so on.  On a release line, there will be a handful of releases. The first
is called .1 (Eucalyptus.1 for example).  Follow-on releases are numbered from
there: Eucalyptus.2, Eucalyptus.3, and so on.

After the .1 release, new releases in a line are made only for severe problems
such as security problems, data loss, or feature breakage.

A new release line is created by making a "release master" branch in each
involved repo.  These are named ``release/RELEASENAME``.  This branch
will be where changes are accumulated to create each release in the line.
Releases will be tagged ``release/RELEASENAME.1``,
``release/RELEASENAME.2``, and so on.

.. note::

  Prior to the Teak release, branches were named as follows:

  * Master branch: ``open-release/RELEASENAME.master``
  * Point releases: ``open-release/RELEASENAME.1``, ``open-release/RELEASENAME.2``, etc

The above applies to components that a release pins by git reference: those that
are deployed from, or built out of, a checkout of their own repository.  Other
components are pinned by published version instead, as a dependency declared in
a manifest belonging to a component that *is* pinned by git reference.  Python
libraries pinned in ``edx-platform``'s requirements files, and NPM packages
pinned in a frontend site's ``package.json``, both work this way.

Components pinned by version are not branched or tagged for the release, and
declare ``openedx.org/release: null`` in their ``catalog-info.yaml``.  The
record of which version a release pins lives with the depending component, on
that component's release branch.  Their own branching and versioning policy,
including how long a given version line keeps receiving fixes, is defined by
the decisions governing their ecosystem; see `Related Decisions`_ below.

Being pinned by version does not reduce a component's obligations under `Levels
of support`_.  A component that is supported in a release must keep publishing
fixes for the version line that release pins, for the life of that release.

Involving repos in the Open edX build process
=============================================

:ref:`Release Data in catalog-info.yaml`
defines annotations the ``catalog-info.yaml`` metadata file that can be used to
indicate that a repo needs to be tagged for releases.

Details can be found in the above ADR but in summary there will be a new catalog
annotation by the name of ``openedx.org/release`` that will indicate whether or
not a given repo should be tagged for Open edX releases.


Transitive Dependencies and Docs Builds
=======================================

Note that transitive dependencies should not be explicitly tagged for release.
If a transitive dependency has a need for their own docs build, this can be
configured in the ReadTheDocs admin panel to build a release with the
appropriate RELEASENAME, build from the tagged release version that is included
in RELEASENAME.

Installing Open edX
===================

The Open edX community provides a supported installation method, `Tutor`_. Tutor
is suitable for both development and production environments.

.. _Tutor: https://docs.tutor.edly.io/


Related Decisions
*****************

The following related decisions modify or enhance this OEP, but have not yet been fully incorporated as updates to this OEP:

.. toctree::
   :caption: OEP-10 Decisions
   :maxdepth: 1
   :glob:

   oep-0010/decisions/*

Change History
**************

2026-08-04
==========

* Distinguish components that a release pins by git reference from those it
  pins by published version.
* `Pull request #815 <https://github.com/openedx/openedx-proposals/pull/815>`_

2025-08-24
==========

* Updated the link to the Google sheets calendar
* `Pull request #731 <https://github.com/openedx/openedx-proposals/pull/731>`_


2025-05-23
==========

* Update the naming convention for release branches
* Remove mention of Devstack as a supported development environment
* Clarify that Tutor is suitable for production environments
* `Pull request #712 <https://github.com/openedx/openedx-proposals/pull/712>`_


2023-09-28
==========

* Reference catalog-info.yaml instead of OEP-2 for where we store
  release metadata.
* `Pull request #526 <https://github.com/openedx/openedx-proposals/pull/526>`_

2022-02-24
==========

* Remove info about older installation methods that are no longer relevant.
* `Pull request #452 <https://github.com/openedx/openedx-proposals/pull/452>`_

2020-04-26
==========

* Added the "maybe" key for "openedx-release".
* `Pull request #145 <https://github.com/openedx/openedx-proposals/pull/145>`_

2018-08-22
==========

* Installation details adjusted to match current Hawthorn realities.
* `Pull request #78 <https://github.com/openedx/openedx-proposals/pull/78>`_

2016-11-21
==========

* Document created
* `Pull request #26 <https://github.com/openedx/openedx-proposals/pull/26>`_
