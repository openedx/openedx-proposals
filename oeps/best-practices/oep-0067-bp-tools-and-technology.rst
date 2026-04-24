.. _OEP-67 Standard Tools and Technologies:

OEP-67: Standard Tools and Technologies
#######################################

.. list-table::
   :widths: 25 75

   * - OEP
     - :ref:`OEP-67 <OEP-67 Standard Tools and Technologies>`
   * - Title
     - Tools and Technology Standards
   * - Last Modified
     - 2026-04-24
   * - Authors
     - Feanil Patel <feanil@axim.org>
   * - Arbiter
     - Braden MacDonald  <braden@opencraft.com>
   * - Status
     - Accepted
   * - Type
     - Best Practice
   * - Created
     - 2023-09-05
   * - Review Period
     - 2023-09-05 - 2023-10-10
   * - Resolution
     - * https://github.com/openedx/openedx-proposals/pull/518
   * - References
     - * Replaces :ref:`OEP-11 Front End Technology Standards`
       * Replaces :ref:`OEP-18 Python Dependency Management`

Abstract
********

The Open edX Platform is built using many first- and third-party tools and technologies.
This document covers the current recommendations for which tools and technologies should be used.

Motivation
**********

The Open edX platform has a lot of first and third party tools and technology that it uses.
The problems we're trying to solve are:

* It is difficult to know the current best practice for a given area.

* It is unclear how to go about the discussion and decision-making process to
  change a best practice.

:ref:`OEP-11 Front End Technology Standards` serves as a great example for the
usefulness of this kind of OEP but limits itself to be relevant only to
frontend technologies.  This OEP supersedes :ref:`OEP-11 Front End Technology Standards`
and encompass not only front-end technology but any
technology that:

* Is considered a best practice for the Open edX Platform

* Is relevant to a large subset of the Open edX Platform

Specification
*************

General Technology Selection
============================

This technology is not specific to frontend or backend code.

#. **Codecov should be used to report code coverage by unit tests**

   **Rationale**: It is important to measure the amount of code covered by our
   automated test suites. By striving for a high level of test coverage, we can
   reduce the number of bugs that can only be found via manual testing,
   and by using codecov to run coverage in CI,
   contributors are automatically reminded to include tests for any new code.

   The Open edX community has standardized around `Codecov`_ as a coverage reporting service.

#. **Use Github actions for CI Testing**

   **Rationale**: As a healthy open source project we need good continuous integration
   testing to prevent the introduction of obvious bugs into the platform code.  The Github Actions
   tool should be used to perform Continuous Integration (CI) testing for all Open edX repositories
   that perform any sort of software testing.

.. _Frontend Technology Selection:

Frontend Technology Selection
=============================

.. note::

   The phrase “frontend” is used to mean any part of the platform that is shown to
   users. This encompasses views rendered in Python on the server, interactive
   interfaces written using JavaScript, and CSS styling.

.. _Use React:

#. **Use React**

   **Rationale**: React must be used for building new UIs, as it is
   widely adopted by the community and strikes a balance between
   flexibility and feature richness.

#. **Use React Query for data loading**

   For loading data from the platform or other external sources (e.g. via REST
   APIs), `React Query`_ should be used.

   For client-side state management, regular React "state" and "context" should
   be used alongside React Query, following the best practices described in the
   official React docs: `Managing State`_.

   **Exception**: Much of our frontend code is currently written using Redux for
   both data loading and state management, and that is also an accepted
   approach. However, for new code or major refactorings of existing code, it is
   recommended to use React Query and plain React state instead as described.

   **Decision Record**: For details, see
   :ref:`OEP-0067-0010 React Query <Use React Query>`.

#. **Use Jest and React-Testing-Library to test React components**

   We use `Jest`_ and `React-Testing-Library`_ to test React components.
   We are deprecating the `Enzyme`_ library.
   New tests must not use Enzyme, and any repositories planning to move to React 18 or newer need to
   replace Enzyme.

#. **Target the latest standardized JavaScript version (ECMA-262)**

   **Rationale**: Open edX JavaScript should be written consistent with the latest
   ECMA-262 specification in order to ensure future support, the largest
   community and the availability of modern features. Currently, ES2022 is
   the latest version with full compatibility from `the browsers we support`_.
   There is no need to use Babel, core-js, nor polyfills to provide backwards
   compatibility with unsupported browsers nor to transform ES2022 code into
   ES6/ES5 code.

   **TypeScript**: TypeScript should be utilized in frontend code wherever it
   seems to provide value (especially in any code that is reused: libraries, API
   wrappers, shared components, data type definitions, etc.). For details, see
   :ref:`OEP-0067-0008 TypeScript <Use TypeScript>`.

   **JSX**: Babel or other tooling may be used to add JSX syntax support to our
   JavaScript/TypeScript code.

   **Porting**: Files written in ES5 and/or plain JS should be gradually
   converted to the newer standard (e.g. TypeScript with ES2022 syntax) as new
   development in those feature areas allows.

#. **JavaScript code should follow the edX ESLint configuration**

   **Rationale**: In order to standardize and enforce Open edX's JavaScript
   coding style across multiple codebases, edX has published an ESLint
   configuration that provides an enforceable specification. The
   `edX ESLint Config`_ is made available as an npm package that can be
   installed into any Open edX package.

#. **JavaScript libraries should be installed via npm**

   **Rationale**: It is important that JavaScript libraries are kept
   up-to-date, and one key component is to make it as simple as possible
   to perform upgrades. Projects that use npm maintain all of their
   library requirements in a single `package.json` file, which can be
   easily changed as the versions change.

#. **JavaScript projects should publish a package-lock file**

   **Rationale**: To keep dependencies up-to-date and builds consistent, JavaScript
   projects should allow patch and minor upgrades in their package.json file and
   commit a package-lock.json file. The package-lock.json file will keep a full list
   of dependencies and their versions, ensuring when built for deployments the
   version of libraries are consistent. This follows the same pattern used in Open edX
   Python code - see :ref:`OEP-18 <OEP-18 Python Dependency Management>` for more information. For more information on
   package-lock files see `Package Lock`_.

#. **Keep dependencies up to date by using Renovate**

   **Rationale**: JavaScript dependencies are updated constantly and can be difficult
   to maintain over time. `Renovate`_ makes this easier by automatically updating
   dependencies in a ``package.json`` file and verifying the tests still work. When
   possible, leverage Renovate to ensure JavaScript software remains up to date.
   Documentation on how to configure Renovate automation on a repository is available
   in :doc:`docs.openedx.org:developers/how-tos/enable-javascript-upgrade-automation`.

#. **JavaScript projects should use caret ranges for npm dependencies**

   **Rationale**: Using caret ranges (e.g., ``"^1.2.3"``) in our ``package.json`` more clearly
   communicates that our projects are designed to accept semver-compatible (minor and patch)
   updates. This intent is important because merely pinning top-level dependencies does not
   extend to their transitive dependencies—potentially leading to duplicative or outdated
   packages in our bundles. Moreover, specifying dependencies with caret ranges streamlines
   our `Renovate`_ workflow. With caret ranges, non-breaking updates are typically resolved
   automatically during installation, so Renovate will primarily flag major version bumps
   that require manual intervention. In contrast, pinned dependencies can trigger frequent
   Renovate updates for every minor or patch change, increasing maintenance overhead.

   **Decision Record**: For details, see
   :ref:`Use Caret Ranges for npm Dependency Versions`.

#. **JavaScript should be bundled using Webpack**

   **Rationale**: `Webpack`_ is the tool of choice for code optimization and
   bundling, as it is widely used throughout the industry, and because
   it seamlessly handles modern code bases as well as all of the older
   technologies used by Open edX, such as `AMD Modules`_. Webpack should be
   implemented to handle as much of the "asset pipeline" as possible,
   rather than passing this responsibility on to Django.

#. **JavaScript dependencies should be managed with ES Modules**

   **Rationale**: JavaScript module systems allow front end code to specify
   its dependencies and be grouped into bundles that minimize the assets
   needed to provide page functionality. The most prevalent module syntax
   is currently `ES2015 Modules`_, which should be adopted everywhere
   JavaScript/TypeScript code is used.

   **Exception**: Some of our existing JavaScript uses the older
   `AMD Modules`_ syntax for modules. AMD Modules are interoperable
   with ES2015 Modules if Webpack is used for bundling, so this is fine in
   legacy code, but should not be used in new or refactored code.

#. **CSS should be generated using SCSS**

   .. note::

      While there are no current plans to deprecate SCSS, the recommendation
      as of February 2025 is to use CSS variables instead of SCSS variables
      wherever possible now that browsers support them directly.

      See :ref:`CSS variables should be used to support runtime theming <use-css-variables>`

   **Rationale**: Sass's SCSS syntax is an extension of CSS that adds power
   and elegance to the basic language. It makes the maintenance of large
   CSS files more manageable though the ability to use variables, mixins,
   imports and more. In particular, it makes theming possible with the
   ability to override variables that define colors, fonts etc.

   You can find out more about Sass in the official `Sass documentation`_.

#. **CSS variables should be used to support runtime theming**

   .. _use-css-variables:

   **Rationale**: SCSS variables are evaluated by the SASS preprocessor, and cannot be
   changed afterwards. CSS variables are evaluated by the browser and
   therefore can be changed at runtime, enabling dynamic theming support.

#. **API calls should be made with the frontend-platform**

   **Rationale**: The `frontend-platform`_ simplifies the process of
   talking to Open edX APIs by using Axios interceptors and handling JWT Cookie
   authentication. It also provides React components to handle private routes
   and should be used when possible. When making calls to non-Open edX APIs
   Axios should be used to provide a consistent API.

   The `fetch`_ API was considered but Axios was chosen for its more
   intuitive API, particularly when handling HTTP errors with rejected
   promises.

#. **Server-side content should be rendered with Django Templates**

   **Rationale**: Although it is advised to use client side templating with
   React, see `Use React`_,  in legacy code that generates HTML on the server,
   Django templates should be used. There are many template languages available
   for Django, but the simplest option is to use the built-in Django template
   engine.

   The Open edX codebase has a mixture of Django and Mako templates, but the
   former are easier to reason about because they don't support arbitrary
   code evaluation.

   **Exception**: Mako templates can continue to be used within edx-platform
   for consistency with the existing code base. This is because the base
   templates and theming support are all provided via Mako, so it is too
   much to expect a new feature to be implemented with a different framework.
   There is much desire to replace Mako within edx-platform so this
   exception may eventually be removed.

#. **BundleWatch should be used to automatically monitor JavaScript bundle sizes**

   **Rationale**: It is important for users of the Open edX platform that we deliver
   reasonably sized JavaScript bundles. This provides faster load times to all users,
   and is vital for users with low bandwidth and/or metered connections. To ensure we
   don't unintentionally increase the size of our JavaScript bundles, we utilize BundleWatch
   for automated bundle size monitoring.

#. **Development dependencies should be separated into devDependencies**

   **Rationale**: To keep installation and deployment of our MFEs as fast as
   possible, the "dependencies" field in ``package.json`` should reference only
   the packages needed to build the bundle of the MFE (this includes webpack via
   frontend-build, as well as any dependencies used in the actual MFE/React
   code). Any dependencies used only for testing, linting, formatting, or other
   development tasks should be put into "devDependencies" (e.g. Jest, eslint,
   TypeScript, ``@types/`` packages, etc.).

Backend Technology Selection
============================

This section will cover any technology or tools that are not considered a part
of the frontend.

.. note::

   This section is incomplete. It should include:

    * Django

    * Django Rest Framework

    * Python?

    * drf-yasg (A thing that is currently standard but that we hope to change)

    * tox

    * pytest

    * edx-lint/pylint

#. **MySQL should be used as the default backing database**

**Rationale**: The Open edX platform has traditionally run on MySQL, and this is
the only officially supported Django ORM database backend. A number of schema
design decisions and optimizations assume that you are running MySQL. That being
said, we are open to patches and fixes that enable other databases, so long as
they do not require disruptive migrations that would break existing MySQL
installations. Support for these other database backends is not guaranteed and
may require plugins or lag behind the official releases.

#. **pyproject.toml should be used to declare project metadata and dependencies**

   **Rationale**: `PEP 621`_ standardized project metadata in ``pyproject.toml``,
   and `PEP 735`_ added dependency groups — a standard way to declare
   context-specific dependencies (test, docs, CI, dev, etc.). Using
   ``pyproject.toml`` as the single source of truth replaces the older pattern
   of ``requirements/*.in`` files and ``setup.py`` / ``setup.cfg``, simplifies
   tooling, and aligns with the broader Python ecosystem.

#. **uv should be used for Python dependency locking and virtual environment management**

   **Rationale**: `uv`_ consolidates dependency resolution, locking, virtual
   environment creation, and tool execution into a single fast tool. It replaces
   `pip-tools`_ (``pip-compile`` / ``pip-sync``) and produces a single
   ``uv.lock`` file from ``pyproject.toml`` dependency groups. The existing
   ``make upgrade`` pattern is preserved — the target now runs
   ``uv lock --upgrade`` instead of ``pip-compile --upgrade``.

   **Decision Record**: For details, see
   :ref:`Use uv for Python dependency management`.

.. _PEP 621: https://peps.python.org/pep-0621/
.. _PEP 735: https://peps.python.org/pep-0735/
.. _uv: https://docs.astral.sh/uv/
.. _pip-tools: https://github.com/jazzband/pip-tools


Decisions
*********

This section lists the decision records for any time a standard technology has
been change in the system along with the reasons why and the alternatives
considered.

.. A set of toctrees to ADR folders with decisions.  We moved all the existing fronted decisions to under this OEP.

.. toctree::
   :caption: Overall Decisions
   :maxdepth: 1
   :glob:

..   oep-0067/decisions/overall/*

.. toctree::
   :caption: Frontend Decisions
   :maxdepth: 1
   :glob:
   :numbered:

   oep-0067/decisions/frontend/*

.. toctree::
   :caption: Backend Decisions
   :maxdepth: 1
   :glob:

   oep-0067/decisions/backend/*



Consequences
************

* :ref:`OEP-11 Front End Technology Standards` will be superseded.

* All ADRs under OEP-11 will be moved to be under this OEP instead.

* :ref:`OEP-18 Python Dependency Management` will be superseded by the backend
  technology recommendations in this OEP.

* Future decisions for technology changes will require an ADR and an update to this OEP

Change History
**************

2026-04-24
==========

* Add ``pyproject.toml`` and ``uv`` to Backend Technology Selection
* Add backend ADR for adopting uv (replacing pip-tools)
* Mark :ref:`OEP-18 Python Dependency Management` as superseded
* `Pull request #784 <https://github.com/openedx/openedx-proposals/pull/784>`_

2025-02-10
==========

* Add note encouraging CSS Variables instead of SCSS
* Remove note about the Tech Radar as it is deprecated (`DEPR ticket <https://github.com/openedx/public-engineering/issues/264>`_)
* `Pull request #675 <https://github.com/openedx/openedx-proposals/pull/675>`_

2025-01-30
==========

* Updated "edX"-specific language to "Open edX" where applicable
* Removed and/or updated author notes from when the OEP was being written
* `Pull request #666 <https://github.com/openedx/openedx-proposals/pull/666>`_

2024-07-25
==========

* Added "Development dependencies should be separated into devDependencies"
* `Pull request #615 <https://github.com/openedx/openedx-proposals/pull/615>`_
* Changed guidance on React state/data loading to recommend React Query instead of Redux
* Updated JavaScript/TypeScript guidance
* `Pull request #616 <https://github.com/openedx/openedx-proposals/pull/616>`_

2023-09-05
==========

* Document created
* `Pull request #518 <https://github.com/openedx/openedx-proposals/pull/518>`_

2024-05-23
==========

* Updated "Keep dependencies up to date" to recommend `Renovate`_ instead of Greenkeeper

.. Cross-references
.. _AMD Modules: https://github.com/amdjs/amdjs-api/wiki/AMD
.. _Babel: https://babeljs.io/
.. _Codecov: https://about.codecov.io/
.. _edX ESLint Config: https://github.com/openedx/eslint-config
.. _frontend-platform: https://github.com/openedx/frontend-platform
.. _enzyme: https://airbnb.io/enzyme/
.. _ESLint configuration cascading: https://eslint.org/docs/user-guide/configuring/configuration-files#cascading-and-hierarchy
.. _ES2015 Modules: http://www.ecma-international.org/ecma-262/6.0/#sec-imports
.. _ES2017: https://tc39.github.io/ecma262/
.. _Fetch: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
.. _Jest: https://jestjs.io/
.. _Managing State: https://react.dev/learn/managing-state
.. _package lock: https://docs.npmjs.com/cli/v6/configuring-npm/package-locks
.. _React: https://github.com/facebook/react
.. _React Query: https://tanstack.com/query/latest/docs/framework/react/overview
.. _React-Testing-Library: https://testing-library.com/docs/react-testing-library/intro
.. _ReactRender: https://github.com/openedx/edx-platform/blob/4b38b1f750918ff83c02cff776681aabe44bd689/common/djangoapps/pipeline_mako/templates/static_content.html#L159-L167
.. _Renovate: https://renovatebot.com/
.. _Sass documentation: http://sass-lang.com/
.. _the browsers we support: https://github.com/openedx/browserslist-config#supported-browsers
.. _Webpack: https://webpack.github.io/
