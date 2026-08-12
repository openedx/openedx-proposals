.. _Merging frontend-base:

Merging frontend-base
#####################

Status
******

Accepted

Summary
*******

This ADR describes how a repository's ``frontend-base`` conversion branch becomes
its main branch, what governs its branches once it has, and where the
:term:`micro-frontend <Micro-frontend>` it replaces goes on living.

Context
*******

Converting a ``frontend-app-*`` repository into an :term:`App Repository`, as
:ref:`ADR 0002 <Frontend Apps>` describes, changes it too deeply to do the work in
place.  Its main branch still builds a micro-frontend that operators are running
and that Open edX releases are cut from, so the conversion happens on a long-lived
``frontend-base`` branch instead, publishing prerelease packages of its own while
it is in flight and taking ports of the main branch's features and fixes as they
land.

Eventually the conversion is done, and the repository should become what the
:term:`Composable Architecture` expects: a publisher of packages composed into a
:term:`Site`, rather than a deployable application.  Three things have to be
settled for that to happen.  How the converted branch takes over.  What governs the
repository's branches afterwards.  And what becomes of the micro-frontend line that
releases were being cut from, which does not stop mattering the day the conversion
finishes.

Decision
********

1. The conversion branch is merged with the "ours" strategy
===========================================================

A conversion branch is not reconciled with the main branch.  The two have diverged
across nearly every file, and there is nothing to reconcile in any case, since the
conversion branch already carries the main branch's features and fixes as ports.
The merge keeps the conversion branch's tree wholesale while still joining the
two histories::

  git switch frontend-base
  git merge -s ours main
  git switch main
  git merge --ff-only frontend-base
  git push origin main

The first merge produces a commit with ``frontend-base``'s tree and both branches
as parents.  The second is therefore a fast-forward, which is the point: the main
branch is never rewritten, so clones, open pull requests, and everything already
pointing into its history survive the merge.  Once it is pushed, the
``frontend-base`` branch has served its purpose and can be deleted.

2. After merging, OEP-10 governs
================================

From that moment the repository is subject to
:ref:`OEP-10 <OEP-10 Open edX Releases>` and its ADRs, with the single exception
section 3 makes.  The main branch is unstable and publishes alphas, ``stable``
carries the current stable major on the "latest" dist-tag, and maintenance
branches are cut as :ref:`ADR 0002 <Frontend Stable Branches>` describes.
Release participation follows :ref:`ADR 0003 <Frontend Release Strategy>`, which
is to say by published version rather than by a branch cut at release time.

3. The micro-frontend line continues on legacy-mfe
==================================================

Immediately before the merge, a long-lived ``legacy-mfe`` branch is cut from the
main branch's tip.  It carries the micro-frontend the conversion replaces, and it
is where any further ``release/RELEASENAME`` branches for that micro-frontend are
cut, should any release still include it.

This is an explicit exception to :ref:`ADR 0003 <Frontend Release Strategy>`.
A converted repository annotates ``openedx.org/release: "legacy-mfe"``, so that
release tooling keeps cutting the micro-frontend from that branch while it
still ships.  The exception lasts only as long as the micro-frontend does: once
no supported release includes it, the annotation goes to null, ADR 0003 applies
unmodified, and ``legacy-mfe`` can be archived.

Consequences
************

Tooling that assumes a repository's release branch is cut from its default
branch will need attention, since for a converted repository the two are no
longer the same.

The main branch's first-parent history becomes the conversion branch's, with
the legacy micro-frontend's history joined on the second parent.  A full ``git
log`` shows both, while path-limited logs and ``git blame`` only apply to the
converted code.

Merging is a one-way door for a given repository.  Nothing prevents further work on
``legacy-mfe``, but the main branch does not go back to building a micro-frontend.

Rejected Alternatives
*********************

**Merge the conversion branch normally.**  A regular merge would try to combine two
trees that no longer resemble each other, across nearly every file, and a
conflict resolution that large is both enormous and pointless: the result wanted is
simply the conversion branch's tree, which the ``ours`` strategy produces directly.

**Force-push the main branch to the conversion branch's tip.**  This is how
:ref:`ADR 0002 <Frontend Stable Branches>` moves ``stable`` onto a new major, and
the objection there does not apply here, since ``legacy-mfe`` would keep the
outgoing commits either way.  What differs is the branch: this is the repository's
default branch, so a non-fast-forward move would invalidate every clone and recompute
every open pull request against unrelated history.  ADR 0002 accepts that cost on
``stable`` because ``stable`` is consumed from NPM rather than checked out, and pays
it repeatedly.  Here it would be paid once, on the branch where it hurts most, so
the ``ours`` merge and its permanent merge commit are the better trade.

References
**********

* `Land frontend-base <https://github.com/openedx/frontend-base/issues/243>`_, the
  issue where the merge mechanism was worked out

Change History
**************

2026-08-12
==========

* Document created
* `Pull request #815 <https://github.com/openedx/openedx-proposals/pull/815>`_
