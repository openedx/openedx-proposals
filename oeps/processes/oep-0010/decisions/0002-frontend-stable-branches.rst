.. _Frontend Stable Branches:

0002. Frontend Stable Branches
##############################

Status
******

**Accepted**

Context
*******

Semantically-released frontend repositories in the Open edX org need a rallying
point where new features integrate: a place where developers have enough leeway
to push the envelope and fix bugs without having the code immediately shipped
to thousands of users.  That branch is commonly called "master" or "main" in
projects that follow the "main is unstable" strategy; others call it "next",
"develop", or "alpha".  What is almost universal is that at least one branch
serves this purpose.

In contrast, main branches here have historically been deemed stable, with every
commit that lands expected to be production ready.  Most repositories have no
"next" branch either.

At the other end of the spectrum, consumers who are not ready for a new major
version of a package still need fixes for the one they are on.  A single stable
branch cannot serve them: once it has moved on, the versions they are running
have nowhere to be patched from.  The structure therefore has to include lines
that outlive the current stable major, each publishing on its own.

What follows settles which branches such a repository has, what each of them
publishes, and how changes travel between them.  Which of those lines a given
Open edX release depends on, and for how long, is settled by
:ref:`ADR 0003 <Frontend Release Strategy>`.

Decision
********

The following decisions apply to any frontend repository in the Open edX org
once it starts publishing to NPM with ``semantic-release``.

1. Main is unstable
===================

A repository's main branch is deemed unstable. This has two major consequences:

1. Using the main branch in production is not supported;
2. The DEPR process does not apply to it; breaking changes can land with no
   warning.

Notably, though, the main branch retains the following properties:

1. New features should be developed for and merged to it before any other
   branch;
2. To facilitate collaboration, it is never rebased.

2. Publication of the main branch
=================================

A repository's main branch is published semantically to NPM on an "alpha"
prerelease tag, with a monotonically incremented number suffix.  For instance::

  frontend-base@2.0.0-alpha.4
  frontend-app-instructor-dashboard@2.0.0-alpha.2

3. Stable and maintenance branches
==================================

Stable code is published from a long-lived ``stable`` branch, which always
carries the newest stable major.  It owns the NPM "latest" dist-tag and is
published semantically, with no breaking changes allowed after publication::

  frontend-base@1.0.5
  frontend-app-instructor-dashboard@1.4.3

When the main branch contains breaking changes over what ``stable`` carries
and is deemed ready for widespread use, its work graduates onto ``stable`` as
the next major (see `4. Graduating main onto stable`_).  To continue maintaining
the previous major, an "n.x" branch is cut from
its last tag, where "n" is that major's number and the ".x" is literal.  For
instance, once ``stable`` has moved from the 1.x line to the 2.x line, a ``1.x``
branch is cut at the last 1.x tag::

  stable    2.0.0, 2.1.0, ...    (dist-tag: latest)
  1.x       1.4.3, 1.5.0, ...    (dist-tag: 1.x)

These maintenance branches are also published semantically, with no breaking
changes allowed.  Each one owns the NPM dist-tag matching its own name, which
is what keeps "latest" pointing at ``stable``. Consumers, for their part,
select a maintained major by semver range::

  "@openedx/frontend-base": "1.x"

An "n.x" branch accepts both new minors and patches within its major.  To patch a
minor that the line carrying it has moved past, whether that line is ``stable``
or an "n.x" branch, an "n.m.x" branch is cut from that minor's last tag, where "m"
is the minor's number.  These are patch-only: a
feature landing on one still results in a patch release.  Continuing the example
above, with 1.5.0 already shipped from ``1.x``, a ``1.4.x`` branch cut at the last
1.4 tag carries on the 1.4 line::

  stable    2.0.0, 2.1.0, ...    (dist-tag: latest)
  1.x       1.5.0, 1.6.0, ...    (dist-tag: 1.x)
  1.4.x     1.4.4, 1.4.5, ...    (dist-tag: 1.4.x)

4. Graduating main onto stable
==============================

Graduation is the moment ``stable`` stops carrying one major and starts carrying
the next.  It cannot be a fast-forward: by then ``stable`` holds cherry-picked
backports that are not on the main branch, so the two have diverged.

Rather than reconcile that divergence, graduation retires the branch.  The
outgoing line keeps its history under its "n.x" name, and ``stable`` is
re-pointed at the main branch's tip::

  git push origin stable:1.x
  git push --force origin main:stable

Cutting the "n.x" branch of `3. Stable and maintenance branches`_ and retiring the
old ``stable`` are therefore the same act.  Since the second push rewrites a
published branch, graduation is a deliberate push by a maintainer rather than a
PR.

5. Backports
============

All changes, including bug fixes and security patches, should target the main
branch first.  Once merged, they can be backported to ``stable`` and to the
appropriate maintenance branches.  This "main first" approach ensures that fixes
are never lost when a new line is cut: they are part of the main branch's
history and will naturally flow into future releases.

The only foreseeable scenario where a change may land on ``stable`` or a
maintenance branch without a corresponding change landing on the main branch
first is when the latter has diverged enough that the fix or feature no longer
applies to it, because the affected code has been removed or rewritten.

The flow, by kind of change:

Breaking change
  Lands on the main branch, and stops there (until the entire main
  branch is deemed stable and subsequently released.)

New feature
  Lands on the main branch.  It may be backported to ``stable`` and to an
  ``n.x`` branch when the new feature is stable and there is a specific reason
  to do so, such as Product demand; not all non-breaking features will be.
  These features stop at ``n.x``.

Bug fix
  Lands on the main branch, and is backported to ``stable`` and to every
  ``n.x`` and ``n.m.x`` branch still under support (see
  :ref:`ADR 0003 <Frontend Release Strategy>`).

Two backport methods are available, in order of preference:

Fast-forward merge
  If no breaking changes have landed on the default branch since ``stable`` was
  last updated, fast-forward it to the current tip.  This brings in all
  intermediate commits, and is the ideal approach early in a release cycle,
  before the two have meaningfully diverged.

Cherry-pick
  When the default branch contains breaking changes that must not reach
  ``stable``, cherry-pick individual merge commits instead, resolving conflicts
  or reworking the change as needed.  This is the most common method once the
  branches have started to diverge.

Regardless of the method, backports are submitted as PRs against the target
branch, and the PR description should reference the original one for
traceability.

Consequences
************

Operators and developers who track a main branch get no warning before a
breaking change, and no DEPR process to lean on.  The alpha dist-tag exists so
that this is an explicit choice rather than an accident.

The cost is branch count and backport work.  Every maintained line is a branch
somebody has to cut at the right tag and backport to, and the number of live lines
grows with how many majors and minors remain under support.  The "main first" rule
keeps this tractable, but it does not make it free.

``stable`` also moves non-fast-forward once per major, so its branch protection
has to leave room for whoever performs a graduation, and anyone holding a clone of
it has to reset when one lands.  No history is lost in the move, since the
outgoing commits keep a branch of their own.

Rejected Alternatives
*********************

**Keep every main branch production ready.**  This is the org's historical
practice, and it leaves concurrent breaking work with nowhere to integrate.  Two
features that need to evolve together, each breaking on its own, either ship to
users before they are ready or are developed in isolation and reconciled late.

**Merge the main branch into stable.**  Resolving in the main branch's
favor with ``-X theirs`` does not actually produce its tree, since that option
only settles conflicting hunks: files changed on ``stable`` alone survive into the
result.  Even done correctly, the merge commit would sit on ``stable`` rather than
on the main branch, leaving ``stable`` ahead of it.  No later backport could then
be a fast-forward, and the main branch would go on deriving its alphas from the
tag ``stable`` carried before graduation, publishing them below the major that
was just released.

**Graduate with an ``ours`` merge onto the main branch.**  Merging ``stable`` into
the main branch with the ``ours`` strategy yields a commit that keeps the main
branch's tree while recording ``stable`` as a second parent, after which
``stable`` fast-forwards onto it and never moves backwards.  It works, but the
main branch pays: it carries a permanent merge commit whose second parent
claims content the strategy discarded, so every backport shows up twice in
``git log``.

**Commit the main branch's tree onto stable as a single commit.**  Resetting
``stable``'s tree to the main branch's and committing keeps ``stable``
fast-forwardable and leaves the main branch alone, but the main branch's commits
never become ancestors of ``stable``.  The new major's release notes collapse to
that one commit, its version turns on how the message is worded rather than on the
history behind it, and because ``stable`` stops being an ancestor of the main
branch, later backports cannot fast-forward and the main branch cannot see the new
major's tag.

Change History
**************

2026-08-04
##########

* Document created
* `Pull request #815 <https://github.com/openedx/openedx-proposals/pull/815>`_
