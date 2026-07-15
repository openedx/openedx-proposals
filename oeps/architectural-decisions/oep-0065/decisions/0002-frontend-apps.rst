.. _Frontend Apps:

Frontend Apps
#############

Status
******

Accepted

Summary
*******

This ADR describes what frontend :term:`apps <App>` are in the :term:`Composable Architecture`, and how the ``frontend-app-*`` repositories are organized around them. It also summarizes, at a high level, how existing :term:`micro-frontends <Micro-frontend>` become apps.

Context
*******

OEP-65 proposes composing the :term:`Open edX Frontend` from independently developed :term:`apps <App>` at build time - into one or more :term:`Sites <Site>` built by a shared :term:`Shell` - as a means to enable build-time composability and :term:`shared dependencies <Shared Dependencies>`. The OEP lays out a series of changes necessary to enable these capabilities, which it refers to as building a "reference implementation".

This reference implementation is effectively a new underlying architecture for our frontend. This ADR refers to this new architecture as the :term:`Composable Architecture`, as opposed to the historical :term:`Micro-frontend Architecture` that has existed prior to OEP-65.

The ``frontend-app-*`` repositories are central to this architecture: rather than being independently deployed applications, they become the home of the :term:`apps <App>` that a :term:`Site` composes. This ADR defines what an app is and how these repositories are organized, and summarizes how existing micro-frontends migrate to become apps.

Please see :ref:`Create a unified platform library` for more information on ``frontend-base``.

Decision
********

The fundamental unit of the frontend is the :term:`App`.

What an App is
==============

An :term:`App` is a self-contained part of the :term:`Open edX Frontend` that is composed into a :term:`Site` at build time. An app usually represents a well-bounded area of the UI - such as courseware, the login page, or account settings - and typically owns one or more routes. Beyond its routes, an app can contribute:

* :term:`Widgets <Widget>` - components rendered into a :term:`Slot`, whether its own, another app's, or the :term:`Shell`'s. The header and footer, for instance, can be overridden with alternate implementations via widgets, and new tabs on the course home are widgets.
* *Services* - implementations of platform services such as logging or analytics.
* *Scripts* - arbitrary scripts attached to the page.

Crucially, there is no separate "plugin" concept. What other frameworks would split into "applications" and "plugins" are, in the :term:`Composable Architecture`, all just apps: an app that contributes a top-level area of the UI and an app that only contributes a :term:`Widget` to another app's :term:`Slot` are the same kind of thing.

App Repositories
================

Apps live in :term:`App Repositories <App Repository>`. These repositories:

* Are always named ``frontend-app-*``, regardless of what their apps do. This is the single naming convention for frontend application code going forward.
* Provide one or more apps. Most provide a single app, but some provide several related apps centered on a domain (learning, authoring, authn, etc.) that share code and dependencies. The question of which apps belong in which repositories, and where the right boundaries are, is beyond the scope of this ADR.

Because plugins are apps, components previously built as ``frontend-plugin-framework`` plugins become apps, and repositories previously named ``frontend-plugin-*`` should be renamed to ``frontend-app-*``. There is no longer a meaningful distinction between a "plugin repository" and an "app repository".

Consequences
************

Migration
=========

Existing :term:`micro-frontends <Micro-frontend>` (and ``frontend-plugin-*`` repositories) will need to migrate to become :term:`App Repositories <App Repository>` built on `frontend-base <https://github.com/openedx/frontend-base>`_. Here are a few essential examples of what needs to happen during the conversion:

* **The shell takes over the foundations.** Apps no longer initialize ``frontend-platform`` or render their own header and footer; the :term:`Shell` provides application initialization, layout, branding, and services. Apps drop the foundational libraries this replaces - ``frontend-build``, ``frontend-platform``, ``frontend-plugin-framework``, ``frontend-component-header``, ``frontend-component-footer``, and ``@edx/brand`` - in favor of ``frontend-base``.
* **Shared dependencies become peer dependencies.** The dependencies the shell provides - React, Paragon, React Router, Redux, and so on - become peer dependencies of the App Repository rather than direct ones, so that a :term:`Site` resolves them to a single shared version at build time.
* **Configuration moves to the Site.** App Repositories no longer carry environment-specific ``.env`` or ``env.config`` files; configuration lives in a :term:`Site Config` (see :ref:`Frontend Sites`). A repository may keep a git-ignored config file for local development.
* **New tooling.** The ``openedx`` CLI from ``frontend-base`` replaces ``fedx-scripts`` from ``frontend-build``, and App Repositories are published to npm so that :term:`Sites <Site>` can consume their apps.

The complete, step-by-step migration process will be documented in a `Frontend App Migration How To <https://github.com/openedx/frontend-base/blob/main/docs/how_tos/migrate-frontend-app.md>`_ in the ``frontend-base`` repository.

As we adopt ``frontend-base``, the libraries it replaces will undergo their own deprecation processes (via the `DEPR process <https://docs.openedx.org/projects/openedx-proposals/en/latest/processes/oep-0021-proc-deprecation.html>`_), coordinated with the migration of the micro-frontends included in Open edX releases. After that deprecation, the :term:`Micro-frontend Architecture` will cease to be supported.

Unsupported Customizations
==========================

The :term:`Micro-frontend Architecture` took an extreme approach to "flexibility", allowing MFEs to diverge from each other in a variety of ways as described in :ref:`OEP-65 <OEP-65 Frontend Composability>`. As a result, in the process of migrating them to the :term:`Composable Architecture`, there could be unforeseen refactoring that may need to happen in some MFEs that don't map into apps well, or which have customizations that aren't supported by the Shell. While we hope to provide enough extensibility mechanisms to reduce the need for forking or hacky customizations, there will be customizations we haven't anticipated, which the community will need to work around or find ways to support.

Consistent Dependency Versions
==============================

Addressing our *lack* of dependency version consistency is one of the primary drivers of OEP-65.

The shell will support specific version ranges of shared dependencies (such as React, Paragon, or React Router). All :term:`apps <App>` composed into the shell's :term:`Site` will be expected to use (or at least be compatible with) that version. We intend to create lock-step version consistency of shared dependencies across all apps in the platform. We envision each Open edX release supporting a particular major version of each shared dependency.

References
**********

* :ref:`OEP-65: Frontend Composability <OEP-65 Frontend Composability>`
* :ref:`OEP-65 Frontend Glossary <Frontend Glossary>`
* :ref:`ADR-0001: Unified Platform Library <Create a unified platform library>`
* :ref:`ADR-0003: Frontend Sites <Frontend Sites>`
* `Frontend App Migration How To <https://github.com/openedx/frontend-base/blob/main/docs/how_tos/migrate-frontend-app.md>`_

Change History
**************

2024-08-28
==========

* Document created
* `Pull request #626 <https://github.com/openedx/openedx-proposals/pull/626>`_

2024-09-13
==========

* Updating the language use to match and reference the frontend glossary.

2026-07-13
==========

* Rewritten and generalized to match the ``frontend-base`` reference implementation, and retitled from "Frontend App Migrations" to "Frontend Apps". The Decision now defines what an :term:`App` is; the migration is summarized under Consequences, with the details deferred to the migration how-to in ``frontend-base``.
