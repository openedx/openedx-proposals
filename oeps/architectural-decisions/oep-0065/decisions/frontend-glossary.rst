.. _Frontend Glossary:

Frontend Glossary
#################

High Level Concepts
*******************

.. glossary::

  App
    A self-contained part of the :term:`Open edX Frontend` that is composed into a :term:`Site`. An app represents a well-bounded area of the UI - such as courseware, the login page, or account settings - and typically owns one or more routes. It declares its own routes, :term:`Slots <Slot>`, providers, configuration, and the :term:`Widgets <Widget>`, services, or scripts it contributes.

    Apps are composed into a :term:`Site` at build time by being imported into its :term:`Site Config`. In the historical :term:`Micro-frontend Architecture`, the ``frontend-app-*`` repositories were independently deployed applications ("MFEs"); in the :term:`Composable Architecture` they become :term:`App Repositories <App Repository>`, each providing one or more apps. Components previously built as ``frontend-plugin-framework`` plugins are likewise converted into apps, each contributing its UI to a :term:`Slot` as a :term:`Widget`.

  App Repository
    A repository (``frontend-app-*``) that provides one or more :term:`apps <App>` in the :term:`Composable Architecture`. Most App Repositories provide a single app, but some provide several related apps - for example, ``frontend-app-learning`` might provide the courseware, course outline, progress, and dates apps, which share code and dependencies specific to their domain. This stands in contrast to how the ``frontend-app-*`` repositories were used in the :term:`Micro-frontend Architecture`, where each was a single deployable application.

  Micro-frontend
    Micro-frontend is an industry standard term for small, composable, independently deployed pieces of a frontend. It has a specific and narrower meaning in Open edX's frontend. Open edX's decoupled frontend architecture has been called the "micro-frontend architecture" since 2018 or so, and the ``frontend-app-*`` repositories, specifically, are referred to as "micro-frontends" or "MFEs" for short. Some might argue it's a misnomer, as many of our MFEs are quite large. Regardless, MFEs in Open edX refer to our independently deployed, siloed frontends which do not share dependencies, and which may contain one or more distinct parts of the overall frontend.

    The :term:`Composable Architecture` supersedes this approach: the ``frontend-app-*`` repositories become :term:`App Repositories <App Repository>` rather than standalone applications.

  Micro-frontend Architecture
    This is the name for the Open edX frontend architecture based on ``frontend-platform``, ``frontend-build``, and independent ``frontend-app-*`` repositories that represent :term:`micro-frontends <Micro-frontend>`. It has been superseded by the :term:`Composable Architecture`.

  Open edX Frontend
    The complete set of interfaces shown to a user of an Open edX instance. It consists of one or more independently deployed :term:`Sites <Site>`.

  Shared Dependencies
    The capability to share a frontend's dependencies (such as React or Open edX's Paragon) between the :term:`apps <App>` composed into a :term:`Site`. Because those apps are composed into a single build, each shared dependency is bundled only once, which has a significant impact on overall download size for frontend clients.

  Shell
    The top-level frontend, provided by the `frontend-base <https://github.com/openedx/frontend-base>`_ library, that composes a :term:`Site`. The shell initializes the application, loads the :term:`shared dependencies <Shared Dependencies>`, renders the layout (including the header and footer), loads the brand, and composes the :term:`apps <App>` declared in the :term:`Site Config`.

  Composable Architecture
    The name for the Open edX frontend architecture based on ``frontend-base``, the :term:`Shell`, :term:`apps <App>`, and :term:`Sites <Site>`. It supersedes the :term:`Micro-frontend Architecture`. Frontends are composed into a :term:`Site` at build time rather than deployed as independent single-page applications.

  Site
    An independently deployed portion of the :term:`Open edX Frontend` that contains an instance of the :term:`Shell`. In practice, a Site renders the header and footer and composes one or more :term:`apps <App>` into a single bundle, defined by its :term:`Site Config`. A frontend may consist of one Site or several independent Sites, depending on an operator's needs. Navigating between Sites involves a full-page refresh and downloading a new set of dependencies.

    A Site is also where an operator checks in the configuration and customizations for a deployable part of their frontend. Operators typically maintain a Site as their own repository (for example, based on ``frontend-template-site``).

  Site Config
    A TypeScript file (``site.config.tsx``) which represents the configuration of a :term:`Site`. It imports the :term:`apps <App>` the Site composes, configures them, and declares the operator's customizations. It must adhere to the ``SiteConfig`` TypeScript interface, and is *runtime* code.

  Slot
    A designated place in the component hierarchy that will accept :term:`Widgets <Widget>`. Slots have documented layout expectations, configuration requirements, and contextual data that they share with the widgets composed into them.

  Widget
    A component composed into a :term:`Slot`. Widgets are how :term:`apps <App>` and operators extend or override parts of a page - for example, adding a new tab to the course home, or replacing the header - without modifying the app that owns the slot. An app contributes widgets to the :term:`Slots <Slot>` of the :term:`Shell` or of other apps.

Migration
*********

.. glossary::

  External Route
    A route in a :term:`Site` that navigates to an external URL rather than composing an :term:`App`. The destination may be another :term:`Site`, or it may just be some other webpage, such as a support portal. External Routes are an important architectural option which allows us to do partial or gradual updates of the :term:`Open edX Frontend` by dividing it into pieces. If a frontend needs to migrate through breaking changes in its dependencies, like React or React Router, we can split it into two :term:`Sites <Site>` and use External Routes to move between them while migrating :term:`apps <App>` from one to the other as they adopt the change. It also allows some frontends to remain as independently deployed, :term:`legacy MFEs <Micro-frontend>` during migration.

Change History
**************

* Document created
* `Pull request #626 <https://github.com/openedx/openedx-proposals/pull/626>`_

2026-07-13
==========

* Rewritten to match the ``frontend-base`` reference implementation: the composable unit is an :term:`App` (not a "module"), operators compose apps into a :term:`Site` (not a "project"), and composition happens at build time. Removed terminology for the dropped runtime module federation approach (Module, Module Federation, Federated Module, Federated Plugin, Remote, Remote Discovery, Guest, Host, Runtime Module Loading) and the Project/Module Project/Site Project concepts. Renamed "Module Architecture" to :term:`Composable Architecture`.
* Renamed "Plugin Slot" to :term:`Slot`. Replaced the "Plugin" term (and the "Imported Plugin" / "IFrame Plugin" subtypes) with :term:`Widget`, the component composed into a :term:`Slot`. Existing ``frontend-plugin-framework`` plugins are converted to :term:`apps <App>`, just as MFEs are; the build-time vs iframe loading distinction is now described in the :term:`Widget` definition.
* Renamed "Linked Site" to :term:`External Route` to match ``frontend-base``, reframing it as a route that points to an external URL rather than a kind of site.
