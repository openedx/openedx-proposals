.. _Frontend Release Strategy:

0003. Frontend Release Strategy
###############################

Status
******

**Accepted**

Context
*******

One question drives this ADR: how does a semantically-released frontend
repository participate in an Open edX release?

:ref:`OEP-10 <OEP-10 Open edX Releases>` pins a repository into a release by
cutting a ``release/RELEASENAME`` branch and tagging it.  That works for
repositories which are deployed from, or built out of, a checkout of
themselves.  But :ref:`OEP-65 <OEP-65 Frontend Composability>` changes what a
frontend repository is: apps are published as NPM packages and composed into a
:term:`Site` at build time, alongside ``frontend-base`` itself.  The Site is the
deployable artifact; everything it is built from is resolved from NPM by version.

For an app repository, cutting a ``release/RELEASENAME`` branch does not survive
that change.  Nothing deploys from such a branch, because an app is no longer
deployed; it is compiled into a Site.  Nothing consumes it either, because a Site
names its apps by version range in ``package.json``, and a version range does not
select a branch.  The branch would be inert: cutting it, or forgetting to, makes
no difference to what an operator installs, because what a release ships is
decided by the Site's lockfile either way.

Participation therefore has to be by published version.
:ref:`ADR 0002 <Frontend Stable Branches>` establishes the lines those versions
come from: a ``stable`` branch carrying the current stable major, and ``n.x`` and
``n.m.x`` branches carrying the ones still maintained.  What remains is which
repositories get branched for a release and which do not, what the branched ones
record, what dependency ranges they carry, and which lines end up pinned and for
how long.

Decision
********

1. Frontend package repositories are not branched or tagged
===========================================================

Repositories whose release artifact is a package published to NPM are not
branched or tagged for Open edX releases.  This covers ``frontend-base``, the
shared libraries, and every app repository built on ``frontend-base``.  Each
declares ``openedx.org/release: null`` in its ``catalog-info.yaml``.

They participate by published version instead.  Which version, and how it is
recorded, follows from the next two sections.

2. frontend-template-site is branched and tagged
================================================

``frontend-template-site`` is branched and tagged in the ordinary OEP-10 way: it
keeps ``openedx.org/release`` pointing at its default branch, and gets a
``release/RELEASENAME`` branch at cutoff.  It is the only frontend repository in
the release that does.

Its ``package.json`` and ``package-lock.json``, on that branch, are the
authoritative record of which frontend versions the release ships.  There is no
separate manifest to maintain.

Operators maintain their own Sites, which are not part of the release and are not
bound by any of this, though one derived from the template will usually want to
follow the same pattern.

3. The release/RELEASENAME branch takes patch-level ranges
==========================================================

By definition, new features shouldn't be added to stable releases. Thus, on
``frontend-template-site``'s ``release/RELEASENAME`` branch, direct dependencies
on apps and on ``frontend-base`` are expressed as x-ranges constrained to specific
feature levels.  The branch then accepts patches from each maintenance line, but
declines minor version updates::

  "dependencies": {
    "@openedx/frontend-app-authn": "1.0.x",
    "@openedx/frontend-app-instructor-dashboard": "1.2.x",
    "@openedx/frontend-base": "1.0.x"
  }

Its main branch is unaffected: it keeps caret ranges, which are a deliberate
choice there, letting new minors arrive without a PR to every ``package.json``.

4. Maintenance branches carry the release's support window
==========================================================

The minor each package is pinned at is whatever was most recently published when
the branch was cut, as captured in ``frontend-template-site``'s lockfile.  Those
minors determine which ``n.x`` and ``n.m.x`` branches have to exist: a release
pinned at 1.4 once the line carrying it has moved on to 1.5 needs a ``1.4.x``
branch to patch from.  Those branches must keep receiving fixes for the life of
the release, which per OEP-10 is until the following release, or about six months.

This is how a package that is not branched for a release still meets OEP-10's
criteria for being *supported* in it.

Consequences
************

App repositories that today annotate ``openedx.org/release: "master"`` flip to
null as they move onto ``frontend-base`` and into Sites.  The release tooling in
``repo-tools`` will stop cutting and tagging branches in them, and should be
checked for anything that assumes every included component has a
``release/RELEASENAME`` branch.

Operators tracking a release branch of their Site receive bug fixes from each
maintenance line without any change to their ``package.json``, and receive
features only by editing it.  This makes taking a feature onto a release branch a
deliberate act, which is the point, but it also means an operator who wants one
must know which minor it landed in.

Every release under support holds its pinned lines open, so the branch and
backport cost scales with how many releases are supported at once.

Rejected Alternatives
*********************

**Cut release branches in frontend package repositories anyway.**  The status quo
for app repositories.  Beyond being inert, it is incompatible with the scheme
:ref:`ADR 0002 <Frontend Stable Branches>` sets out, in which every branch owns a
version line and each version is derived from history.  A branch named for a
calendar release owns no version line, so anything published from it would land in
a line another branch already owns.  A dist-tag marks a release's version without
needing a branch at all.

**Have frontend-template-site depend on an app's branch rather than a version.**
The other way to give a branch effect, and incompatible with consuming these
repositories as published packages.  A release pins the artifact on the registry;
a branch is source that was never published, so it carries no real version.

**Pin exact versions on the release branch.**  This would require a PR to
``frontend-template-site`` for every patch of every app it composes, and buys
nothing: patches are non-breaking by definition, and the lockfile already records
exactly what a given build resolved.

**Keep caret ranges on the release branch.**  A caret lets the lockfile pick up a
new minor with no change to ``package.json``, so a released branch can start
shipping features nobody assessed against its backend, and can do so without any
reviewable change to the repository.

Change History
**************

2026-08-04
##########

* Document created
* `Pull request #815 <https://github.com/openedx/openedx-proposals/pull/815>`_
