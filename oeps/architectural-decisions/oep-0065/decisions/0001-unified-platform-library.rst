.. _Create a unified platform library:

Create a unified platform library
#################################

Status
******

Accepted

Summary
*******

This ADR proposes combining a number of repositories into a single, unified frontend platform library as a migration strategy for implementing :ref:`OEP-65 <OEP-65 Frontend Composability>`.  This library is named `frontend-base <https://github.com/openedx/frontend-base>`_.

Context
*******

OEP-65 proposes building the :term:`Open edX Frontend` from independently developed :term:`apps <App>`, all composed into a shared :term:`Shell` in one or more :term:`Sites <Site>`, as a means to enable build-time composability and :term:`shared dependencies <Shared Dependencies>`. The OEP lays out a series of changes necessary to enable these capabilities, which it refers to as building a "reference implementation".

This reference implementation is effectively a new underlying architecture for our frontend. This ADR refers to this new architecture as the :term:`Composable Architecture`, as opposed to the historical :term:`Micro-frontend Architecture` that has existed prior to OEP-65.

The bulk of the work to build OEP-65's reference implementation is related to the libraries that MFEs (as opposed to :term:`Apps <App>`) currently depend on:

* frontend-build
* frontend-platform
* frontend-plugin-framework
* frontend-component-header
* frontend-component-footer

It will also involve the creation of a new :term:`Shell` to initialize the frontend and compose :term:`apps <App>` into a :term:`Site`. Prior to this decision, we might have considered putting the shell in its own repository as well:

* frontend-app-shell

There are several related concerns which influence our strategy for migrating these library repositories to the new module architecture, and which influence where we decide to put the shell.

Significant API changes
=======================

The creation of the shell and the move toward composing :term:`apps <App>` is expected to significantly change the API surface across our various libraries.

``frontend-platform``
---------------------

As the shell takes responsibility for much of what ``frontend-platform`` provides to MFEs in the :term:`Micro-frontend Architecture`, many of the APIs it exposes will become internal to the shell. We will also likely need to add new APIs for interactions between the shell and apps.

``frontend-build``
------------------

The development environment will also change, as we want to be able to run a single development server that loads the MFE under development into the shell. This means creating alternatives to frontend-build's existing helpers to support the new architecture.

``frontend-component-header`` / ``frontend-component-footer``
-------------------------------------------------------------

The header and footer will be owned by the shell and need not be libraries any more. A caveat to this is, of course, that we need to maintain good ways to customize them.

``frontend-plugin-framework``
-----------------------------

We expect ``frontend-plugin-framework`` to be folded into ``frontend-base`` to unify it with the rest of the runtime library, as they will be intimately tied to each other going forward. We'll also be creating new APIs around composing :term:`widgets <Widget>` into :term:`slots <Slot>`.

``frontend-app-*`` Repositories
-------------------------------

While we don't expect to merge these into ``frontend-base``, they are the consumers of all the API changes above. We expect it will be simpler for developers to absorb these changes - and the inevitable bug fixes - by updating a single library dependency, rather than trying to navigate an interconnected dependency tree with cascading version updates.

Reducing Dependency Maintenance
===============================

One of the motivators of OEP-65 was reducing dependency maintenance. Not explicitly called out in the OEP is the desire to have fewer repositories to manage, which will in turn further reduce the number of dependencies in our frontend. We see value in combining the library repositories listed above into a single library, which will in turn reduce the number of dependencies to manage in downstream repositories.

Modernization
=============

We also feel the need to continue to modernize our library repositories by adopting industry standard technologies like TypeScript, or more performant webpack loaders, and there's some sentiment that this may be the right time to make these changes as we're already undergoing a paradigm shift. ``frontend-platform``, for instance, simplifies significantly if we use TypeScript types instead of the bespoke "interface" and "service implementation" system in that repository.

Deprecation of the micro-frontend architecture
==============================================

As described in OEP-65, we're migrating from the :term:`Micro-frontend Architecture`, where an MFE is a standalone, independently deployed site, to a :term:`Composable Architecture` where the MFEs become :term:`apps <App>` composed into a common :term:`Shell` which owns the header, footer, initialization, and shared dependencies.

One way or another, this is a paradigm shift that will involve breaking changes and migration work on behalf of community members. We would like to provide a clear a path forward for operators, developers, and maintainers to adopt the module architecture. We believe that a clean break and a cohesive and clear platform for modules is more approachable than a more granular deprecation of certain features and code in the existing libraries.

Decision
********

We will migrate our existing foundational libraries to the new :term:`Composable Architecture` by creating a unified platform library - `frontend-base <https://github.com/openedx/frontend-base>`_ - to act as the primary dependency of frontend :term:`apps <App>`.

This library will include the relevant features of ``frontend-build``, ``frontend-platform``, ``frontend-plugin-framework``, ``frontend-component-header``, and ``frontend-component-footer``, as well as a new :term:`shell <Shell>` application, along with new features and capabilities necessary to implement the vision in OEP-65. We will also opportunistically fold in ``eslint-config``, which sees very little development but is its own source of dependency management overhead.

This new library will be released as one npm package with the following responsibilities.

Runtime library
===============

The library will export a subset of the APIs in ``frontend-platform``, along with ``frontend-plugin-framework``'s API and new APIs specific to the composable architecture and the shell. This library will be a (peer) dependency of frontend apps.

Development tool configurations
===============================

The library will provide base ESLint, Jest, TypeScript, and Webpack configurations. ESLint, Jest, and TypeScript will be similar to what ``frontend-build`` provides.

The webpack configurations will support a variety of build targets for the new :term:`Composable Architecture`, as well as app configurations to enable us to migrate to the new unified platform library in a backwards compatible way. The new build targets include:

* A production configuration suitable for building an :term:`App` into a deployable :term:`Site`.
* A release configuration which packages an :term:`App Repository`'s :term:`apps <App>` as an npm package for build-time inclusion in a :term:`Site`.
* A development server configuration to serve an :term:`App` inside a :term:`Shell`, complete with hot module reloading.

These webpack configurations will be exposed via a function like ``frontend-build``'s '``createConfig`` so that they can be modified and extended as necessary by the MFEs.

CLI tool
========

Similar to ``fedx-scripts`` provided by ``frontend-build``, the ``frontend-base`` library will provide a CLI tool. This tool will expose commands to run the above webpack configurations, as well as other commands related to building and developing :term:`Sites <Site>`. The rationale and details of these CLI commands are the subject of a future ADR.

Consequences
************

This approach allows us to treat the :term:`Composable Architecture` as an independent effort, as opposed to a set of features that need to fit in with the existing :term:`Micro-frontend Architecture`. We greatly reduce or eliminate the risk of regressions in libraries that the entire community relies on while we build a replacement which resembles and borrows code from them, but is otherwise significantly different. This division makes it explicitly clear which features are used by the old architecture and which are used by the new one.

Further, it means we can migrate to the new architecture in a backwards compatible way by atomically porting MFEs to :term:`Apps` that use the new `frontend-base <https://github.com/openedx/frontend-base>`_ library, again, without running the risk of destabilizing existing libraries with multiple breaking changes over time.

We believe this is a reasonable migration strategy given the generally low rate of feature development in these libraries. We incur incrementally more maintenance burden in the interim while both sets of libraries exist, and a level of effort in absorbing any features or bug fixes in the existing libraries. However, we believe that will be offset by an increase in development velocity for the new library, a clearer deprecation process, less risk of regressions in existing code, and an easier mental model of how the architecture is changing.

References
**********

* :ref:`OEP-65: Frontend Composability <OEP-65 Frontend Composability>`
* :ref:`OEP-65 Frontend Glossary <Frontend Glossary>`
* :ref:`ADR-0002: Frontend Apps <Frontend Apps>`
* :ref:`ADR-0003: Frontend Sites <Frontend Sites>`

Change History
**************

2024-06-26
==========

* Document created
* `Pull request #598 <https://github.com/openedx/openedx-proposals/pull/598>`_

2026-07-13
==========

* Updated terminology to match the ``frontend-base`` reference implementation: build-time composition of :term:`apps <App>` into :term:`Sites <Site>` via the :term:`Shell`, in place of runtime module federation. Renamed "Module Architecture" to :term:`Composable Architecture`.
* Retitled from "Create a unified platform repository" to "Create a unified platform library" (and renamed the file accordingly), since ``frontend-base`` is a library.

2024-09-13
==========

* Updating the language use to match and reference the frontend glossary.
