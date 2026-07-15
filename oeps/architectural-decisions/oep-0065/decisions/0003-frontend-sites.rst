.. _Frontend Sites:

Frontend Sites
##############

Status
******

Accepted

Summary
*******

This ADR describes the architecture of and rationale for frontend :term:`Sites <Site>`, a way of building and customizing the :term:`Open edX Frontend`.

Context
*******

OEP-65 proposes constructing the :term:`Open edX Frontend` from independently developed :term:`apps <App>` into one or more :term:`Sites <Site>` as a means to enable composability and :term:`shared dependencies <Shared Dependencies>`. The OEP lays out a series of changes necessary to enable these capabilities, which it refers to as building a "reference implementation".

This reference implementation is effectively a new underlying architecture for our frontend. This ADR refers to this new architecture as the :term:`Composable Architecture`, as opposed to the historical :term:`Micro-frontend Architecture` that has existed prior to OEP-65.

As part of this re-architecture, we need a way for operators to work with and deploy the :term:`Shell`, which is a wrapper around our existing micro-frontends that will provide application initialization, the header and footer, and the services (logging, analytics, i18n, etc.) provided by ``frontend-base``'s runtime library. With micro-frontends, operators are expected to check out the MFE, add a configuration file (``.env`` or ``env.config.js``) co-located with the code, and run the build/development commands within the repository.

If we carried over this paradigm to the Shell, that would imply that operators should check out frontend-base, add a configuration file to their checkout, and run the CLI commands within the frontend-base repository.

In both cases, there are several important developer experience issues with this approach:

* A developer or operator should not have to check out the source code of a platform or framework in order to work with it.
* Adding a configuration file to the source code could tacitly encourage forking when it isn't necessary, as operators would like somewhere to check in their configuration.
* There's no clear path or best practice for operators to make customizations beyond simple configuration file changes.
* The approach of adding a config file and customizations to the source code is at odds with industry best practices and developer expectations when working with nearly any popular framework, such as Django, React, Next.js, or Rails.

Finally, our documentation and best practices around how to build and customize the Open edX frontend have always been a bit of an afterthought, and operators have been left to "figure it out" and rely on tools like Tutor to manage the complexity and uncertainty for them. We believe that the platform itself should take more responsibility for providing best in class tools and patterns for working with the code.

Decision
********

We will create a well-supported, built-in configuration and customization framework by adding support for frontend :term:`Sites <Site>`.

A frontend :term:`Site` is a repository, created and owned by an operator, that contains all of the frontend configuration and customized :term:`apps <App>` for one independently deployable part of their Open edX frontend. An operator will often have just one Site. An operator may have multiple Sites for a given Open edX frontend if they want to deploy it in multiple, independent pieces.

A Site uses a :term:`Site Config` file to define its configuration. It renders a header and footer and composes one or more :term:`apps <App>` into a single build.

A Site consists of:

* One or more configuration files which account for all of an operator's config and customizations across all their environments.
* A set of build targets expressed as ``"scripts"`` in ``package.json`` which point at ``openedx`` CLI commands from ``frontend-base``.
* (Optional) A ``src`` sub-folder containing the operator's custom :term:`apps <App>` and extensions.

The :term:`apps <App>` composed into a Site are imported into its :term:`Site Config` and bundled with the Site at build time. They may come from the Site's own ``src`` folder or from :term:`App Repositories <App Repository>` the Site depends on.

.. image:: ../site-project-architecture.png

To describe the steps in the above image:

1. A build is started with the ``npm run build`` command, which references a ``scripts`` entry in the Site's package.json.
2. That script delegates to the ``openedx`` CLI ``build`` command provided by ``frontend-base``.
3. The ``build`` CLI command runs webpack with the ``build`` webpack config.
4. Webpack uses the :term:`Shell` - in ``frontend-base`` - as its entry point.
5. The Shell initialization code imports the :term:`Site Config` (i.e., ``site.config.build.tsx``) file from the Site.
6. The :term:`Site Config` file imports any :term:`apps <App>` from :term:`App Repositories <App Repository>` it depends on (defined as ``dependencies`` in package.json), along with any other apps from the ``src`` sub-folder.

`frontend-template-site <https://github.com/openedx/frontend-template-site>`_ is a reference implementation of a Site: operators can copy it as a starting point and customize it to their needs.

Implicit Sites
==============

Fundamentally, a Site consists of:

* A :term:`Site Config` file.
* Appropriate build scripts which use ``openedx`` CLI commands.
* Optionally, the source code of :term:`apps <App>` to bundle into the Site (either in-repository or as dependencies).

This means that any repository that satisfies these requirements can act as a Site. These are *implicit* Sites.

Of particular note, ``frontend-app-*`` repositories will satisfy these requirements if we add :term:`Site Config` files to them. In fact, we anticipate this will be a desirable way to do local development on :term:`App Repositories <App Repository>`.

Consequences
************

The addition of Sites creates a first class way of managing the configuration and customization of an Open edX frontend instance without checking out the source of the Open edX Platform frontend itself.

As we begin to migrate the frontend to the :term:`Composable Architecture`, operators will need to adjust their development, build and deployment processes to use Sites. While this will require some effort, we believe that focusing the customization of the Open edX frontend around Sites is a clearer, more approachable paradigm that has significant precedent in the industry.

We expect that there will be edge cases that we didn't anticipate in the Composable Architecture, particularly around customization, which may still require operators to fork the source, but we should endeavor to minimize cases where that's necessary.

References
**********

* :ref:`OEP-65: Frontend Composability <OEP-65 Frontend Composability>`
* :ref:`OEP-65 Frontend Glossary <Frontend Glossary>`
* :ref:`ADR-0001: Unified Platform Library <Create a unified platform library>`
* :ref:`ADR-0002: Frontend Apps <Frontend Apps>`
* `frontend-template-site <https://github.com/openedx/frontend-template-site>`_ - a reference implementation of a Site

Change History
**************

2024-09-04
==========

* Document created
* `Pull request #626 <https://github.com/openedx/openedx-proposals/pull/626>`_

2024-09-13
==========

* Updating the language use to match and reference the frontend glossary.

2026-07-13
==========

* Rewritten to match the ``frontend-base`` reference implementation. The "Project" concept (Site Projects and Module Projects) is replaced by :term:`Sites <Site>`, and the module federation "Module Project" deployment path is dropped in favor of composing :term:`apps <App>` into a Site at build time. Renamed the ADR from "Frontend Projects" to "Frontend Sites".
