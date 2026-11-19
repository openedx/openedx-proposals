.. _OEP-10 Open edX Releases:

OEP-10: Open edX Releases
#########################

+---------------+---------------------------------------------------+
| OEP           | :ref:`OEP-10 <OEP-10 Open edX Releases>`          |
+---------------+---------------------------------------------------+
| Title         | Open edX Releases                                 |
+---------------+---------------------------------------------------+
| Last Modified | 2026-08-26                                        |
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

Open edX is packaged into releases: a rolling release that is updated
continuously as components earn their way into it, and named releases that are
frozen from it, cut when they are ready, and then supported for a window
announced when they ship.  This document details the process.


Motivation
**********

Named Open edX releases are cut when they are ready rather than on a schedule,
and the rolling release is updated weekly.  This document standardizes aspects of
the release process to ensure that all involved understand and participate
appropriately.


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


Release tiers
=============

Open edX code moves through three tiers, each carrying a different guarantee.

**Master branches** are where work lands first.  A change there is gated only by
its own repository's CI, and nothing is promised about how it interacts with any
other component.  The project undertakes nothing about master, and nobody should
run it in production.

The **rolling release** is the set of components currently known to work
together.  A component enters it by promotion, earned against the criteria in
`Promotion`_ below and confirmed by an integration gate run against the whole
set.  It is snapshotted and tagged weekly.  Only the most recent snapshot is
supported; older snapshots remain fetchable and receive nothing.  There is always
a tested upgrade path from the previous snapshot to the current one.  Operators
who want fixes and features as they become available run the rolling release.

A **named release** is a freeze of the rolling release, cut when it is ready as
described in `Release creation`_ and supported for twelve months from its ``.1``
tag.  It receives fixes and no features.  Operators who want a stable target, and
who would rather upgrade once a release than once a week, run a named release.


Promotion
=========

A commit of a component enters the rolling release by **promotion**, which is
earned rather than scheduled.  Promotion is the only way in, and it rests on two
things that this OEP fixes:

- the component's own CI is green at that commit, and no release-critical bug
  (see `Release-critical bugs`_ below) is attributed to the component; and

- the set that results from taking that commit passes the integration gate.

Components are not promoted one at a time.  Eligible commits are assembled into a
candidate set, and the integration gate runs against that set: Tutor builds and
boots it, the end-to-end suite passes against the running deployment, and an
instance running the current snapshot upgrades to it cleanly, migrations
included.  What passes or fails is the set, so changes that span repositories are
promoted together or not at all.

Everything else about eligibility is deliberately left open here, and is to be
settled in a decision record of its own: whether a commit has to age on master
before it can be promoted and for how long, what is required of a component that
publishes a package before its consumers can be bumped, how a candidate set is
assembled, and what the gate has to contain beyond the three checks above.  Those
are operational questions whose right answers depend on how the gate performs in
practice, and they are expected to be revised more than once without reopening
this document.  Until such a record is accepted, the criteria above are the whole
of the requirement.

Maintainers own the promotion of their own components.  A maintainer may hold a
component at a given commit, which needs no justification, and may waive any
waiting period the promotion criteria impose in order to get an urgent fix out.
Both are recorded alongside the manifest.

Because the upgrade path from one snapshot to the next is guaranteed,
contributions have to observe constraints that do not apply to master on its own.
A migration may not be removed until it has shipped in a published snapshot.  A
breaking change lands together with a shim that survives at least one named
release.  Data migrations are forward-only, and idempotent from the previous
snapshot.  Settings, toggles, and public APIs are deprecated under
:ref:`OEP-21 <OEP-21 DEPR>` for one full named release before removal.

A component that fails the gate, or that has an unresolved release-critical bug,
is held at its last passing commit while the rest of the set moves on.  A
component that has been held for two consecutive months is removed from the
release set, which is a matter of removing its ``openedx.org/release`` annotation
and is announced as a deprecation.


Release-critical bugs
=====================

A bug is **release-critical** if, in a supported configuration, it makes the
platform unusable or unbuildable for a significant segment of operators, has a
security impact, loses or corrupts data, breaks the documented upgrade path from
the previous release or the previous snapshot, or is a regression against the
previous release in a documented feature.  Everything else is an ordinary bug,
however annoying.  An unfinished feature is not a bug at all.

Release-critical bugs carry the ``release-critical`` label in the repository of
the component they are attributed to, and the union of those labels across the
org is published as a single public query.  The Build-Test-Release Working Group
owns triage of that queue, in consultation with the maintainer of the component a
bug is attributed to.  Security bugs under embargo are tracked by the Security
Working Group and reported as a count and a component, so that they can block
without being disclosed by the act of blocking.

A release-critical bug blocks the promotion of the component it is attributed to,
and blocks a release cut.  No other class of work does.


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

- It must have been promoted into the rolling release, per `Promotion`_ above,
  before the release's freeze date.

- It must be installable in at least one of the supported installation methods.

- It must be usable in the installed configuration.

The last two are established by the integration gate that promotion runs, rather
than asserted: a set that Tutor cannot build and boot is not promoted, and so
cannot be in a release.

Beyond inclusion, to be **supported** in an Open edX release, a component must
meet further criteria:

- It must be documented thoroughly and accurately.

- A maintainer (or maintainer group) must agree to be
  minimally available for answering questions in the public channels.

- A maintainer (or maintainer group) must agree to provide fixes for
  release-critical bugs attributed to the component, as defined in
  `Release-critical bugs`_ above.

Many of these criteria are open to interpretation, or varying degrees of
effort. For example, what does "documented thoroughly" mean? We can't quantify
that. It and other loose criteria are included here because they are important
parts of providing a finished quality product, and we don't want to overlook
them.


Dependencies
============

Our software is built atop other software layers supported by their creators.
It is important to consider the support windows for those layers when choosing
which version to use.  A named release is supported by the community for twelve
months from its ``.1`` tag, as described in `Support windows`_ below.  The
supporting layers must be supported by their developers for that entire window.

Typically this means choosing Long Term Support (LTS) versions of the supporting
layers, but it's possible shorter-term support versions will provide the support
needed.  Choose with headroom: a version is picked well before the freeze, and
the interval from there to the ``.1`` tag is not fixed in advance, so a layer
whose own support ends soon after our window is due to close is a layer we will
end up carrying ourselves.

The layers in question here are Django, Python, and Ubuntu.  Here's a `calendar
of the known support windows`__ and how they overlap with Open edX plans.

.. __: https://docs.google.com/spreadsheets/d/1wtpoypH1XOPc_G6h9AUNXJ6XiNKD6dlkMP3lubdpE9I


Release creation
================

Release lines are named with words in alphabetical order: Dogwood, Eucalyptus,
Ficus, Gingko, and so on.  On a release line, there will be a handful of
releases.  The first is called .1 (Eucalyptus.1 for example).  Follow-on releases
are numbered from there: Eucalyptus.2, Eucalyptus.3, and so on.

There is no fixed interval between release lines.  What is scheduled is the
freeze: the freeze date for the next line is announced at least three months in
advance, and once announced it does not move.  The cut date is not announced at
all, because it depends on what the freeze finds.

On the freeze date, the current state of the rolling release is copied to a
release candidate: everything promoted by then is in the release, and everything
not promoted by then is not.  From the freeze until the cut, the candidate
accepts fixes for release-critical bugs and nothing else, promoted through the
ordinary gate.

The release is cut when every release-critical bug attributed to the candidate is
either fixed or explicitly waived.  Nothing else moves that date: not a feature
that wants a few more weeks, and not the calendar.  Work that missed the freeze
is already available in the rolling release, and ships in the next named release.

A freeze that is not converging is a problem to be worked, not a deadline to be
met.  From the fourth week of a freeze, the Build-Test-Release Working Group
publishes the outstanding release-critical list and what it intends to do about
each entry, at least every two weeks, and may at any point waive a bug or drop
the component it is attributed to from the release set.  A freeze ends because
that list is empty, not because time has passed.

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
components are pinned by published version instead, as a dependency declared by
a component that *is* pinned by git reference.  Python
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


Point releases and stable updates
=================================

Every merge to a ``release/RELEASENAME`` branch is tagged automatically, named
for the most recent point release plus the date::

  release/teak.1+2026.08.26

A second tag on the same day takes a counter, ``release/teak.1+2026.08.26.2``,
and once ``teak.2`` is tagged, later ones are named from it.  These are **stable
updates**.  A stable update carries only the testing its branch runs: it says
that this is the release branch, with this fix, at this moment, and that the
branch's CI passed.  It exists so that an operator can take a specific fix as
soon as it lands, and so that a fix can never be merged to a release branch and
remain unavailable.

Point releases - ``.2``, ``.3``, and so on - remain the artifacts the project
stands behind, gated the way ``.1`` is.  They consolidate stable updates that
have already been published, rather than being the first delivery of any of them.
A point release is cut when there are stable updates not yet consolidated into
one, no sooner than a week after the newest of them, and no less often than every
two months while any remain.

There is no such thing as an urgent point release, because urgency is served by
the stable update tag that already exists.  Distributions of Open edX, Tutor
included, pin a stable update tag when they need a fix ahead of the next point
release, instead of carrying the patch themselves.  Patching platform source
inside a distribution remains available for the case where a fix cannot be merged
to the release branch at all, and is otherwise unnecessary.


Support windows
===============

A release line is supported for twelve months from its ``.1`` tag.  The end date
is computed when the line is created and published on the release's page before
the release ships, and it does not move afterwards, in either direction: not when
the following release slips, and not when it arrives early.  An operator knows
the date they are planning against from the day they install.

Because named releases are cut when they are ready rather than on a schedule, the
number of lines in support at a given moment is not fixed.  Every line is
supported for its announced window whatever the others are doing, so a slower
cycle means fewer lines overlap and a faster one means more.

Overlap is not free.  Each supported line is another branch that backports have
to reach, another set of point releases to cut, and another set of dependency
versions whose upstream support has to hold.  Two lines in support is a healthy
overlap, and gives an operator room to plan an upgrade rather than perform one
under time pressure.  Beyond that, the maintenance cost is real and lands on the
same maintainers who are working on the current line, which is a reason not to
cut named releases in quick succession: operators who want changes sooner have
the rolling release, and do not need a new named release to get them.

When a line reaches the end of its window, a final point release is cut if the
release branch carries stable updates that no point release has consolidated, and
the branch is then archived.  Nothing is left merged but unreleased when a branch
stops being supported.

The rolling release has no support window in this sense.  Only its most recent
snapshot is supported, and the way to stay supported is to take the next one.


Involving repos in the Open edX build process
=============================================

:ref:`Release Data in catalog-info.yaml`
defines annotations the ``catalog-info.yaml`` metadata file that can be used to
indicate that a repo needs to be tagged for releases.

Details can be found in the above ADR but in summary there will be a new catalog
annotation by the name of ``openedx.org/release`` that will indicate whether or
not a given repo should be tagged for Open edX releases.

The composition of the release is recorded in a **release manifest**: a file in a
dedicated repository, listing every component the release pins by git reference,
each resolved to a commit.  The manifest is generated from the
``openedx.org/release`` annotations, so those annotations remain the only place
membership is declared; what the manifest adds is the resolved commit for each
member and the history of how it changed.  A state of the release is a commit in
that repository, and the difference between two states is a diff.

Components pinned by version are not in the manifest, and their versions are not
restated there.  They are determined transitively, by reading the files of the
components that are pinned by git reference.

The frontends are the worked example.  ``frontend-template-site``'s
``package.json`` declares the frontend set as its dependencies, and
``package-lock.json`` resolves that declaration to exact versions; between them
they are already a manifest for that part of the release, in the shape a manifest
wants: a declaration of what is included, and a lock that says which versions
were tested together.  So the release manifest names a commit of
``frontend-template-site`` and stops there, and the frontend half of the bill of
materials follows from that commit's lockfile.

The Python components have the same shape one level down - a ``.in`` file
declares, a compiled ``.txt`` locks - but no single component aggregates the
others the way ``frontend-template-site`` aggregates the frontends.  Services are
deployed from checkouts of their own repositories, so nothing declares which
version of a service a release contains except the manifest's git reference to
it.  That is why the manifest pins git references at all, and it is a gap rather
than a design: an arrangement for Python equivalent to the frontend one, so that
the manifest can shrink to the components that genuinely have no publishable
artifact, is worth finding and is not settled here.

Promotion is a change to the manifest, and cutting a release consumes one.
Tagging tooling takes a manifest as input rather than recomputing the set of
included repositories at cut time.


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

2026-08-26
==========

* Add the rolling release as a tier between master branches and named releases,
  entered by promotion and gated by integration testing of the whole set.  The
  detailed promotion criteria are left to a decision record of their own.
* Define release-critical bugs, and make them the only thing that blocks a
  promotion or a release cut.
* Drop the six-month release cadence: schedule the freeze, and cut the release
  when its release-critical list is empty.
* Fix the support window at twelve months from the ``.1`` tag, announced up
  front and independent of when the following release ships.
* Add stable update tags between point releases, and describe the release
  manifest.
* `Pull request #816 <https://github.com/openedx/openedx-proposals/pull/816>`_

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
