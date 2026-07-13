.. _OEP-65 Frontend Composability:

OEP-65: Frontend Composability
##############################

.. list-table::
   :widths: 25 75

   * - OEP
     - :ref:`OEP-65 Frontend Composability`
   * - Title
     - Frontend Composability
   * - Last Modified
     - 2026-07-13
   * - Authors
     - * Adolfo R. Brandes <adolfo@axim.org>
       * Pedro Martello <pedro@hammerlabs.net>
       * David Joy <david@joy.engineering>
       * Brian Smith <bsmith@axim.org>
   * - Arbiter
     - Adam Stankiewicz <astankiewicz@2u.com>
   * - Status
     - Accepted
   * - Type
     - Architecture
   * - Created
     - 2024-04-03
   * - Review Period
     - April 15, 2024 - May 10, 2024
   * - Resolution
     - Slack discussion on merging as Provisional: https://openedx.slack.com/archives/C1L370YTZ/p1713215512929479
   * - References
     - * `FC-0054 - Composable Micro-frontends Discovery <https://openedx.atlassian.net/wiki/spaces/COMM/pages/4063821827/FC-0054+-+Composable+Micro-frontends+Piral+Discovery>`_
       * `FC-0007 - Modular MFE Domains Discovery <https://openedx.atlassian.net/wiki/spaces/COMM/pages/3614900241/CLOSED+FC-0007+-+Modular+MFE+Domains+Discovery>`_
       * :ref:`ADR 0001 - Create a unified platform library <Create a unified platform library>`
       * :ref:`ADR 0002 - Frontend apps <Frontend Apps>`
       * :ref:`ADR 0003 - Frontend sites <Frontend Sites>`
       * :ref:`OEP-65 Frontend Glossary <Frontend Glossary>`

.. contents::
	   :local:
	   :depth: 3

Abstract
********

This OEP proposes that the Open edX frontend adopt :term:`shared dependencies` and build-time composability - implemented by composing our frontends into a shared :term:`shell` application at build time - as an approach to improve the consistency, performance, and flexibility of the frontend architecture.

Motivation
**********

Micro-frontends were originally designed to avoid some of the limitations of the monolithic ``edx-platform`` frontend, namely that otherwise independent teams in edX's large engineering organization were beholden to the build, test, and release lifecycle of the rest of the codebase. This dramatically slowed down the pace of frontend feature development, experimentation, and innovation on the master branch where the rate of change is particularly high.

As a result, the Open edX :term:`"micro-frontend" architecture <Micro-frontend Architecture>` focused on creating an un-opinionated, loosely coupled set of applications with the goal of enabling teams to iterate quickly and independently. This goal was successful.

However, we've discovered that the completely siloed nature of these MFEs has created its own set of problems. Our frontends are more like independent single-page applications than `micro-frontends <https://micro-frontends.org>`_, as we never invested in ways of composing them together. The problems inherent in siloed MFE architecture are described below.

Consistency
===========

Because each MFE is completely independent, this creates inconsistencies from MFE to MFE. Inconsistent installed dependency versions manifest in a variety of ways for users, but we also see inconsistency resulting from MFE developers making divergent choices in isolation.

Reusable components (Paragon)
-----------------------------

MFEs may use different versions of `paragon <paragon_>`_, resulting in functional and stylistic differences.

Header/Footer
-------------

MFEs may use different versions of the `frontend-component-header <frontend-component-header_>`_ and `frontend-component-footer <frontend-component-footer_>`_ components, also resulting in functional, stylistic, and content/navigation differences. MFE authors may also make their own headers and footers in isolation without following the best practice of using the shared components.

Branding
--------

Brand packages created from `brand-openedx <brand-openedx_>`_ may be different versions, resulting in any number of subtle visual differences. MFE authors may also make divergent choices like varying page widths, to the detriment of our user experience.

Other Dependencies
------------------

MFEs may have completely different versions of any other dependency. We mitigate some of this by consolidating some important dependencies in `frontend-build <frontend-build_>`_ and `frontend-platform <frontend-platform_>`_, but even those can have different versions from MFE to MFE. For developers, this increases cognitive load and slows velocity because of the need to adjust to the idiosyncrasies of each application.

User and Developer Experience
=============================

Bundle Size
-----------

Each MFE bundles all of its own dependencies, even if they're the same version as another MFE. This means that as a user navigates between MFEs, they end up downloading common dependencies over and over again. This results in megabytes of overhead for the average user navigating between a few MFEs and slows down the entire experience.

Full Page Refreshes
-------------------

Each MFE has its own ``index.html`` page, and needs to load all its own dependencies from scratch whenever you navigate to it. This means that the browser performs a full page refresh each time a user transfers from MFE to MFE.

Build time
----------

Each MFE bundles all its own dependencies at build-time, creating significant overhead for the Webpack build process. When building multiple MFEs, this additional, repeated overhead adds up quickly, making builds prohibitively slow for developers and site operators alike.

Dependency Maintenance
----------------------

Since each MFE has its own complete set of dependencies, the overhead of keeping them all up to date can be overwhelming. Security patches, bug fixes, new features, and breaking changes all add up and create a significant maintenance burden.

Composability
=============

A siloed MFE architecture does not provide a clean, low-overhead way of composing components from multiple MFEs into a single page, or extending an MFE with additional functionality.

The reality of MFEs is that while each application seeks to represent a single cohesive `domain <https://martinfowler.com/bliki/DomainDrivenDesign.html>`_ or `bounded context <https://martinfowler.com/bliki/BoundedContext.html>`_, sometimes content and functionality from more than one domain are needed on the page at the same time.

Alternately, site operators may want to show different *versions* of MFEs to different users while keeping the rest of the app (header, navigation, other MFEs) unchanged.

There are alternatives to build-time composability and :term:`shared dependencies` which are used in some situations. These are not *rejected* alternatives, and so we include them here to help illuminate how their limitations are motivation for adopting build-time composability and shared dependencies.

Shared Libraries (Alternative #1)
------------------------------------

Because each MFE is siloed from each other - both in repositories and at runtime - we can share code by extracting it into a library and having our MFEs depend on it. This creates more repository/dependency overhead.

The :term:`shell` proposed here is itself a shared library that every frontend depends on, so this proposal *builds on* the shared-library approach rather than rejecting it. The difference is what it adds. A shared library on its own only shares *source*: each consuming MFE still builds and deploys independently, so it bundles its own copy of that source, may pin a different version of it, and still renders as a separate, siloed page - leaving the `Bundle Size`_, `Dependency Maintenance`_, and `Full Page Refreshes`_ problems untouched. The shell goes further by *composing* those libraries into a single build and deployment, with one authoritative set of shared dependencies resolved once and rendered on a single page.

Build-time package overrides (Alternative #2)
------------------------------------------------

NPM and ``package.json`` allow site operators to override dependency resolution by installing an alternate version of a dependency prior to build-time. This has historically been how we've allowed operators to override the header, footer, and brand.

The system is confusing and brittle. If a site operator needs different headers/footers/brands for different instances, this multiplies the number of required build processes for an instance.

Frontend Plugins (Alternative #3)
------------------------------------

`frontend-plugin-framework <frontend-plugin-framework_>`_ gives us the ability to share components across MFEs as plugins, either at build-time (direct plugins) or runtime (iframe plugins).

Direct plugins are the same kind of build-time composition this proposal is built on - in the :term:`shell`, apps and :term:`widgets <Widget>` are composed much as direct plugins are. On their own, though, FPF's direct plugins only inject a component; each MFE stays a separate build and deployment, with its own dependencies, header, footer, and page. The shell makes this composition the whole architecture rather than an add-on, so that composing also brings a single shared dependency set and one page. This proposal therefore absorbs and generalizes direct plugins rather than rejecting them.

Iframe plugins are good for sandboxing and isolating code, but they're a very inefficient way to compose a UI, especially given the consistency and user/developer experience concerns raised above. In a way, they exacerbate the problem even more.

Specification
*************

Our approach centers on enabling build-time composability and :term:`shared dependencies`. Together, these two capabilities address the majority of the motivating problems described above (Consistency, User and Developer Experience, and Composability).

We intend to enable these by composing our frontends into a shared :term:`shell` application at build time. Further, we need to complement this new architectural approach with ways of *maintaining dependency consistency* across frontends or we won't be able to realize the benefits of sharing dependencies.

Capability: Build-Time Composability
====================================

The capability to compose content from independently developed apps into the same page - without iframes - solves many of the `Composability`_ and `User and Developer Experience`_ issues above. In particular, it gives us a way of composing UI elements from different domains into a single :term:`site`, and of extending pages via :term:`slots <Slot>`, without each app having to build and deploy its own copy of the surrounding page, header, footer, and shared dependencies. It also cuts down on the number of full page refreshes experienced by users, since navigating between apps composed into the same site no longer reloads the page.

Apps are composed into a :term:`site` at build time by importing them into the site's :term:`Site Config`. This keeps apps decoupled at the source level - a ``frontend-app-*`` repository need not know which sites compose it - while still producing a single, cohesive bundle in which those apps share one set of dependencies.

This composability rests on two ideas: apps are composed into the :term:`shell`, and each app can contribute :term:`widgets <Widget>` into named :term:`slots <Slot>` - its own, or those of other apps or the shell. This is how a page is extended or overridden without modifying the app that owns it, and it unifies and extends the capabilities of the `frontend-plugin-framework <frontend-plugin-framework_>`_ (FPF), whose feature set this proposal absorbs into the :term:`shell`. Existing FPF plugins are converted to apps, much as MFEs are.

Advantages: Build-Time Composability
------------------------------------

* Reduces the frequency of full page refreshes. Independently deployed MFEs today mean that navigating between them loads a completely new page (even if they share dependencies). Apps composed into a single :term:`site` share one page load.
* Improves composability across domains. We have no way to show more than one MFE on the same page today except by using iframes or by creating hard dependencies between MFEs at build-time by extracting 'shared' code into a new library, like `frontend-component-header <frontend-component-header_>`_, `frontend-component-footer <frontend-component-footer_>`_, or `frontend-lib-content-components <https://github.com/openedx/frontend-lib-content-components>`_. Each of these increases our dependency maintenance burden significantly.
* Improves extensibility by giving us :term:`slots <Slot>` and configurable apps, letting operators extend and override pieces of a page through their own :term:`Site <Site>` configuration rather than by forking the frontends being extended.

Capability: Shared Dependencies
===============================

Sharing dependencies between apps complements build-time composability.

MFEs have a number of dependencies which are common between them but which aren't actually shared in any way. The capability to share these dependencies - such as ``react``, `paragon <paragon_>`_, etc. - would mitigate a great deal of our `Consistency`_ and `User and Developer Experience`_ issues.

We expect the following packages - which are used in the vast majority of MFEs today - should all be shared between MFEs.

.. list-table::
   :widths: 50 50

   * - **Package**
     - **Estimated Size**
   * - @edx/brand
     - Variable
   * - @edx/frontend-component-footer
     - 88.1k
   * - @edx/frontend-component-header
     - 156.9k
   * - @edx/frontend-platform
     - 355.3k
   * - @openedx/paragon
     - ~950k
   * - classnames
     - 0.8k
   * - prop-types
     - 0.9k
   * - react
     - 6.4k
   * - react-dom
     - 130.2k
   * - react-redux
     - 11.2k
   * - react-router
     - 58.9k
   * - react-router-dom
     - 77.1k
   * - redux (@reduxjs/toolkit)
     - 3.7k
   * - core-js
     - 241.1k
   * - regenerator-runtime
     - 6.6k

Total size: ~2,087.2k unzipped (Note that these sizes are solely based on bundlephobia.com's estimates, which may not accurately represent how much code we're actually bundling)

Advantages: Shared Dependencies
-------------------------------

* When apps use the same version of a given dependency we see many benefits: significant reduction of developer cognitive load and context switching involved in working with multiple apps, fewer visual inconsistencies, and more. The `Approach: The Shell`_ section has more details on how we foresee this working.
* Reduces runtime bundle size. We'll ship far less code to the client across a user's browsing session.

Note: "dependency maintenance"
------------------------------

Because apps are composed into a :term:`site` at build time, each shared dependency is bundled only once per site - there are no per-app fallback copies to resolve at runtime. This is what reduces bundle size and build overhead within a site.

Sharing dependencies does not, however, reduce *dependency maintenance* on its own. Each ``frontend-app-*`` repository still declares its own dependencies in its own ``package.json``, and those version ranges must stay compatible with the :term:`shell` and with each other so they resolve to a single shared version at build time. Keeping them in sync is not something the build can do for us and is addressed separately in `Approach: Maintaining Dependency Consistency`_ below.

Approach: The Shell
===================

Our approach is to introduce a shared :term:`shell` application through which our frontends are composed. The shell is responsible for initializing the application, loading shared dependencies, rendering the layout (including the header and footer), and composing one or more apps into a single :term:`site`.

Rather than each ``frontend-app-*`` repository building and deploying itself as an independent single-page application, these repositories become collections of composable apps. An operator assembles a :term:`site` whose :term:`Site Config` imports the apps it wants and configures them. The build produces a single bundle in which all composed apps share one set of dependencies.

In terms of Open edX frontends, this means:

#. ``frontend-app-*`` repositories are published as collections of composable apps rather than as standalone applications.
#. A :term:`Site Config` declares which apps a :term:`site` composes and how they are configured.
#. The shell's build tooling - which replaces `frontend-build <frontend-build_>`_ - provides a common set of shared dependencies used by all composed apps.
#. Because everything is built together, each shared dependency is bundled once and apps are wired to the same instance, with no per-app duplication.
#. Pages are extended through :term:`slots <Slot>`, into which :term:`widgets <Widget>` are composed at build time.

Approach: Maintaining Dependency Consistency
============================================

This proposal fundamentally changes how we work with frontend dependencies, and will require us to adopt a more rigorous approach to ensuring dependency consistency and compatibility across apps. Independent app codebases must be kept in sync with regards to dependency versions or we lose the benefits of shared dependencies. Consistency doesn't come for free just by composing apps into a single build.

Apps composed into a :term:`site` must declare compatible versions of the shared dependencies, or the build will resolve them to multiple copies (or fail outright), negating the benefits. The process, tooling, and/or code organization necessary to provide that consistency is not something the build can do for us and needs to be addressed separately.

We expect that this may need to take a number of possible forms.

Process
-------

We need to ensure maintainers and developers know what dependency versions to use, and when they need to upgrade to stay consistent. Open edX release documentation should include information on which frontend dependency versions are compatible with the release, likely pinned to a major version (i.e., React 17.x, Paragon 22.x, etc.)

Further, we recommend that each Open edX release have a single supported major version of all shared dependencies, and that all MFEs be upgraded to it prior to release.

We also need a process to migrate Open edX repositories through breaking changes in third-party dependencies. Ideally following the `Upgrade Project Runbook <https://openedx.atlassian.net/wiki/spaces/AC/pages/3660316693/Upgrade+Project+Runbook>`_.

The architecture allows for migrations through breaking changes in third-party dependencies by running a frontend as separate :term:`sites <Site>` connected by :term:`External Routes <External Route>`, migrating apps from one to the other incrementally as they adopt the breaking change.

Best Practices
--------------

We need to ensure we minimize breaking changes in our own libraries (such as `paragon <paragon_>`_, `frontend-component-header <frontend-component-header_>`_, `frontend-component-footer <frontend-component-footer_>`_, `frontend-platform <frontend-platform_>`_, `frontend-build <frontend-build_>`_, etc.) We suggest accomplishing this by:

* Creating new versions of components with breaking changes (``ButtonV2``, ``webpack.dev.config.v2.js``) rather than modifying existing ones.
* Leveraging the `DEPR process <depr-process_>`_ for communication and removing old component versions.
* Aligning that removal and the subsequent breaking changes with Open edX releases, and documenting it in their release notes.

Further, we could reduce the overhead of dependency maintenance and ensure MFEs stay up to date by pinning dependencies to major versions using ``^`` in our package.json files.

Tooling
-------

Maintainers and developers should be warned of incompatibilities created by their PRs, or outside the repository by another project (such as the shell application).

This could take the form of Github tooling which notifies maintainers and developers that their frontend code has:

#. Drifted behind the compatible version of a shared dependency for a given Open edX release or the main branch.
#. Has upgraded beyond what is compatible with a given Open edX release or the main branch.

Code Organization
-----------------

We may want to refactor how we organize our code to help MFEs ensure they are utilizing dependency versions that align with what other MFEs are using. The goals of such a refactoring are to:

#. Reduce the number of individual dependency updates necessary in MFEs, which in turn reduces maintenance burden.
#. Provide MFEs with a set of shared dependencies guaranteed to be the same as the shell application.
#. Provide MFEs with a more predictable update cycle for shared dependencies, in-line with the Open edX Release cadence.

An ADR attached to this OEP will describe the final approach taken to solve this problem.

Out of Scope
============

There are a few important - but tangential - concerns which are considered out of scope for this OEP and its resulting reference implementation.

* Implementation details of the :term:`shell`'s slot and plugin mechanism.
* How `Tutor <https://docs.tutor.edly.io/>`_ and other distributions will need to change to adopt the shell.
* Opinions on which dependencies we should adopt going forward (such as ``redux`` or other state management solutions)

Rationale
*********

The majority of the concerns expressed in the `Motivation`_ section revolve around a lack of shared dependencies and the way in which MFEs are currently siloed from each other, preventing us from creating a more seamless, cohesive experience.

Composing our frontends into a shared :term:`shell` at build time specifically addresses these use cases. It gives us shared dependencies and composability while keeping our frontend code decoupled at the source level in independent ``frontend-app-*`` repositories, and it can be adopted incrementally (more on that below).

An approach to maintaining dependency consistency is essential to realize the benefits of sharing dependencies. Without it, we've accomplished very little even though we've added the capability. An approach to providing this consistency is not a prerequisite for adopting the shell, to be clear, but the *success* of the shell is tightly coupled to it.

Backward Compatibility
**********************

We intend to maintain backwards compatibility while migrating to and adopting the :term:`shell`. Existing, independently deployed MFEs can continue to run as separate :term:`sites <Site>` reached via :term:`External Routes <External Route>` - a full page refresh - while they are converted into collections of composable apps one at a time. This lets us adopt the shell incrementally rather than in a single cut-over.

Ultimately these frontends will no longer be responsible for initializing `frontend-platform <frontend-platform_>`_ or rendering the header and footer; the :term:`shell` provides these. We will follow the `DEPR process <depr-process_>`_ for retiring that code as each frontend is converted and its apps are composed into a :term:`site`.

Reference Implementation
************************

The reference implementation of this architecture is `frontend-base <https://github.com/openedx/frontend-base>`_, which provides the :term:`shell` and replaces `frontend-build <frontend-build_>`_, `frontend-platform <frontend-platform_>`_, `frontend-plugin-framework <frontend-plugin-framework_>`_, `frontend-component-header <frontend-component-header_>`_, and `frontend-component-footer <frontend-component-footer_>`_.

In frontend-base, an operator assembles their frontend as one or more :term:`sites <Site>`. A :term:`Site Config` (``site.config.tsx``) imports the apps to compose, along with the operator's own customizations, and the shell builds them into a single bundle. Pages are extended through :term:`slots <Slot>`.

The details of how operators structure, configure, and deploy their :term:`sites <Site>` are described in the ``frontend-base`` documentation and the OEP-65 ADRs.

Apps
====

Each ``frontend-app-*`` repository exports one or more apps that a :term:`site` can compose. For instance, ``frontend-app-profile`` exports a profile app. Repositories may also export :term:`widgets <Widget>` to be composed into :term:`slots <Slot>`.

The Shell
=========

The :term:`shell` is the top-level frontend that composes a :term:`site`. It is responsible for:

* Initializing the application.
* Loading the default, expected version of all our shared dependencies.
* Rendering the layout of the application, including the header and footer.
* Loading the brand.
* Composing the apps declared in the :term:`Site Config`, and rendering :term:`widgets <Widget>` into their :term:`slots <Slot>`.

Rejected Alternatives
*********************

Module Federation and Runtime Module Loading
============================================

Earlier iterations of this OEP proposed enabling :term:`shared dependencies <Shared Dependencies>` and composability through `webpack module federation <https://webpack.js.org/concepts/module-federation/>`_, which would have let independently built and deployed bundles be loaded and composed at runtime. This was the original mechanism the OEP was written around.

We ultimately rejected it. None of the problems in the `Motivation`_ section - consistency, bundle size, full page refreshes, build time, dependency maintenance, or composability - actually require runtime module loading to solve; composing apps into a single build at build time addresses them all. Module federation would add significant complexity (a runtime resolution layer, remote discovery, fallback copies of shared dependencies, sandboxing and error boundaries around remotely loaded code) in exchange for a capability our goals don't depend on. Given that, we chose to simplify the architecture and leave it out.

This is not a permanent door-closing. If a concrete need for runtime module loading emerges - for example, a federated marketplace of apps that operators can install and load at runtime without rebuilding their :term:`Site` - module federation is the natural way to provide it, and we would revisit this decision at that point.

Piral
=====

A prior iteration of this OEP and discovery effort (`FC-0007 <https://openedx.atlassian.net/wiki/spaces/COMM/pages/3614900241/CLOSED+FC-0007+-+Modular+MFE+Domains+Discovery>`_) came to the conclusion that we should adopt Piral, a comprehensive micro-frontend web framework, to address our concerns with the Open edX micro-frontend architecture.

After further investigation and review of our stated pains, observed deficiencies, hopes, and vision for Open edx micro-frontends, we chose to adjust course away from Piral. Piral solves composability and shared dependencies in a broadly similar spirit to our shell approach - and can in fact use tools like Webpack internally - but does so in a more proprietary, opinionated, and opaque way, adding additional layers/wrappers around it. We prefer an approach that builds on the tools we already use and stays closer to standard, widely adopted patterns for composing frontends.

Piral is an impressive piece of software, built primarily by one individual, trying to solve a much broader problem than we have. Because of this, it brings along with it a great deal of complexity that we don't need and already have solutions for. Piral aims to be a complete toolkit for building web applications, including authentication, plugins, its own global state mechanism, extensions that provide ready-made UI components, etc.

We need a mechanism to provide shared dependencies and composable frontends that can fit in with our existing ecosystem. Adopting Piral would likely involve significant refactoring of existing MFEs to fit into its framework and to turn them into "pilets", which locks us in to the Piral way of doing things.

It feels like our needs more closely align with the narrower scope of a shared build-time shell, and that it's a more right-sized solution to our architectural problems.

Combining MFEs into 2-3 monoliths
=================================

Folding our micro-frontends together into a few larger frontends (LMS and Studio, for instance) solves our need for shared dependencies in a different way - it just shares all the code so there's one set of dependencies for all of it. We could continue to rely on frontend-plugin-framework for cross-domain plugins, but "plugins" within the larger domain become a simple import from another part of the application.

This approach was abandoned because we still believe that MFE independence is a core need for our platform and we can't go back to a few monolithic frontends. MFE independence continues to allow independent teams to operate with autonomy, lets operators customize, build, and deploy their frontends independently as needed, and creates a more approachable platform for the community by keeping our frontends decoupled and focused.

The build-time :term:`shell` approach can look similar at first glance - it, too, produces a single bundle - but it differs in a crucial way: our frontends remain independent ``frontend-app-*`` repositories that are *composed* into a :term:`site` at build time, rather than being folded together into a few large, co-located codebases.

Combining MFEs into a monorepo
==============================

A monorepo would co-locate all of the MFEs and frontend libraries in the core product in the same repository, but maintain their independent release and deployment cycles. We believe this would help us more readily keep consistent dependency versions across MFEs. But it would also introduce a layer of complexity to our code organization and be a highly invasive way of solving our dependency consistency issues, as we'd have to move all of our core product frontend code into a new repository.

Further, it wouldn't solve our consistency problems for anyone working with custom MFEs or libraries. We want to create parity between the process for core product and non-core product repositories to ensure our approach is serving everyone's needs, not just maintainers of official repositories.

We acknowledge that there are benefits here, but believe that it's more work than it's worth, is only a partial solution, and we have less complex options available to us.

Doing Nothing
=============

We feel that the siloing of micro-frontends, the proliferation of dependencies, the difficulty of extending our platform, and the toil of ongoing maintenance is untenable. This requires us to act to improve the approachability of our frontend architecture; it's not good enough yet.

.. _frontend-platform: https://github.com/openedx/frontend-platform
.. _frontend-build: https://github.com/openedx/frontend-build
.. _frontend-component-header: https://github.com/openedx/frontend-component-header
.. _frontend-component-footer: https://github.com/openedx/frontend-component-footer
.. _paragon: https://github.com/openedx/paragon
.. _brand-openedx: https://github.com/openedx/brand-openedx
.. _frontend-plugin-framework: https://github.com/openedx/frontend-plugin-framework
.. _depr-process: https://docs.openedx.org/projects/openedx-proposals/en/latest/processes/oep-0021-proc-deprecation.html
.. _frontend-template-application: https://github.com/openedx/frontend-template-application
.. _LucidChart source: https://lucid.app/lucidchart/8c2db108-7c14-4525-8e3a-d2853db68b9e/edit?invitationId=inv_7a61f692-df0b-465b-8ec1-5a18ce4447ca

Related Decisions
*****************

The following related decisions modify or enhance this OEP, but have not yet been fully incorporated as updates to this OEP:

.. toctree::
   :caption: OEP-65 Decisions
   :maxdepth: 1
   :glob:

   oep-0065/decisions/*

Change History
**************

2026-07-13
==========

* Reframed the OEP around build-time composability via the :term:`shell` (as implemented in ``frontend-base``). Runtime module loading and webpack module federation - the originally proposed mechanism - were dropped from the reference implementation in favor of composing our frontends into a single build. The composable unit is now referred to as an "app", and operators compose apps into a :term:`site`.
* Removed the "Vite Module Federation" rejected alternative and the runtime-federation architecture diagram, both of which only made sense under the module federation approach. Added a "Module Federation and Runtime Module Loading" rejected alternative explaining why the originally proposed mechanism was dropped and under what circumstances it might be revisited.
* Renamed "Plugin Slot" to :term:`Slot` and introduced :term:`Widget` for the component composed into a slot. Existing frontend-plugin-framework plugins are converted to :term:`apps <App>`, much as MFEs are.

2024-07-31
==========

* Adding a rejected alternative for Vite module federation.

2024-05-13
==========

* Merging OEP-65 as Provisional.

2024-04-03
==========

* Document created
* `Pull request #575 <https://github.com/openedx/openedx-proposals/pull/575>`_ contains all review feedback.

2024-06-26
==========

* Adding a reference to ADR-0001, which describes creation of a unified platform repository.

2024-09-13
==========

* Updating language to match OEP-65's ADRs and leverage the frontend glossary.
* Adding a recommendation to standardize on a single major version of shared dependencies in a given Open edX release.
