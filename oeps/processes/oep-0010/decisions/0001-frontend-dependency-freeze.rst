.. _Frontend Dependency Freeze:

0001. Frontend Dependency Freeze
################################

Status
******

**Accepted**

Context
*******

:ref:`OEP-65 <OEP-65 Frontend Composability>` was written in an attempt to identify and mitigate some of the issues stemming from the siloed development of frontend apps.  Many of these issues are caused by apps using multiple versions of the same dependencies, resulting in reduced consistency, maintainability, user experience (UX) and developer experience (DX).  While frontend-base provides a technical framework to deduplicate these dependencies, it does not make it  impossible to configure a site with some duplicate dependencies.  We must decide on a pattern that establishes how dependency upgrades happen, in such a way that maximizes consistency, maintainability, UX, and DX.

Decision
********

A new release development milestone is hereby established, dubbed the "Frontend Dependency Freeze".  It is fixed at eight weeks prior to cutoff milestone, at which point a decision is published by the `Frontend Working Group`_ regarding what feature level of the following fundamental libraries, as represented by either a specific major or specific minor `semantic version`_ of the corresponding NPM packages, will be used by all frontend repositories that are part of the subsequent Open edX release:

.. _Frontend Working Group: https://openedx.atlassian.net/wiki/spaces/COMM/pages/3090056949/Frontend+Working+Group
.. _semantic version: https://docs.npmjs.com/about-semantic-versioning

- `Node.js`_ (through `.nvmrc`_)
- `Webpack`_
- `React`_
- `React Router`_
- `React Query`_
- `Paragon`_
- `Frontend Base`_
- `Frontend Build`_, `Frontend Platform`_, and `Frontend Plugin Framework`_ (for apps not yet built on Frontend Base)

.. _Node.js: https://nodejs.org/en/download
.. _.nvmrc: https://github.com/nvm-sh/nvm
.. _Webpack: https://www.npmjs.com/package/webpack
.. _React: https://www.npmjs.com/package/react
.. _React Router: https://www.npmjs.com/package/react-router
.. _React Query: https://www.npmjs.com/package/@tanstack/react-query
.. _Paragon: https://github.com/openedx/paragon
.. _Frontend Base: https://github.com/openedx/frontend-base
.. _Frontend Build: https://github.com/openedx/frontend-build
.. _Frontend Platform: https://github.com/openedx/frontend-platform
.. _Frontend Plugin Framework: https://github.com/openedx/frontend-plugin-framework

(This list may be modified in the future to reflect changes in the technology stack, such as those stemming from changes to :ref:`OEP-67 <OEP-67 Standard Tools and Technologies>` or the introduction or removal of an Open edX library.)

The decision will be posted in the appropriate public channels, including creating corresponding issues in all repositories that need to be in sync, so that by the time of the cutoff all aforementioned repositories' main branches must be using the corresponding versions.

The dependency versions this ADR concerns itself with will be configured with a ``^`` (caret) prefix as ``peerDependencies`` in all relevant ``package.json`` files.  Only a major version needs to be specified.  For example::

  ...
  "peerDependencies": {
    "@openedx/frontend-base": "^1",
    "@openedx/paragon": "^23",
    "@tanstack/react-query": "^5",
    "@types/react": "^18",
    "react": "^18",
    "react-router": "^6",
  }

Consequences
************

Once the Dependency Freeze is announced, frontend maintainers must abide by it for the duration of a given release cycle.  Speficically, this means that on all main branches, corresponding dependencies should not be bumped beyond the defined versions from the moment of the announcement until immediately after the cutoff.  And on release branches, the same applies for the life of the branch.  This constraint will not only give developers, testers, and ultimately, users, a firmer base to work off from, but also a more consistent one both visually and under the hood.

Minor and patch version updates are not foreseen to be problematic, as dependency versions will be configured with a caret prefix. This means that whenever a new compatible version of a library is published, it ends up being used automatically by all apps at build time as a result of NPM's deduplication algorithm. In other words, there's no need to issue a PR to every single ``package.json`` just because a bug was fixed in Paragon.

However, making explicit minor version bumps in frozen dependencies - even ones that would be pulled in automatically anyway - is still discouraged.  For instance, an app adding a feature after the freeze that requires a new Paragon component - one that by definition must also have been added after said freeze - is a clear indication of a failure in the additional planning the new milestone affords.  In other words, the right time to coordinate development of new features is *before* the freeze: it's the main reason it's being added in the first place!

Which brings us to the major foreseen cost associated with this change: an increased need for coordination across repositories during a release's development cycle. Frontend maintainers, as part of the Frontend Working Group, need to not only agree on what versions to target, but also, in the case of first-party codebases such as Paragon and Frontend Base, which features to develop and make available for a particular release.  This will need to happen continuously, as the eight weeks before cutoff are intended to be used to bring stragglers up to speed, rather than encompass full development cycles.  In other words, development of new frontend applications targeting a particular release should not wait to start until after the Dependency Freeze; the team should consult with the Frontend Working Group as to which versions are expected to be frozen, and based development on those.

Rejected Alternatives
*********************

The major rejected alternative is the complete absence of a stable set of major frontend dependencies for upcoming releases, which is what we have prior to acceptance of this ADR, with all the cons established in the Context section above.

Otherwise, we reject a smaller set of frozen dependencies.  The proposed list is considered to be the smallest one that still achieves the goals of consistency, maintainability, UX, and DX.  Notably, not freezing Paragon can lead to significant UX consistency issues, and the freezing of Frontend Base is a fundamental technical requirement for all apps built on it - not to mention the benefits we get from de-duplication.  The latter, in turn, necessitates the freezing of the other dependencies such as React itself, which doesn't support having multiple versions running on the same web page.

We also reject a larger, more prescriptive set of dependencies.  While more may eventually be added to the list, developers are, in principle, free to add other requirements to individual package.json files, even if different apps end up using different versions of the same dependency.

Change History
**************

2025-11-20
##########

* Document created
* `Pull request #753 <https://github.com/openedx/open-edx-proposals/pull/753>`_
