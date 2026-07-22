.. _OEP-45 Configuring and Operating Open edX:

OEP-45: Configuring and Operating Open edX
##########################################

.. list-table::
   :widths: 25 75

   * - OEP
     - :ref:`OEP-45 Configuring and Operating Open edX`
   * - Title
     - Configuring and Operating Open edX
   * - Last Modified
     - 2026-07-03
   * - Authors
     - Bill DeRusha <bill@edx.org>
   * - Arbiter
     - Felipe Montoya <felipe.montoya@edunext.co>
   * - Status
     - Needs Revision
   * - Revision Ticket
     - https://github.com/openedx/openedx-proposals/issues/587
   * - Type
     - Architecture
   * - Created
     - 2020-04-06
   * - Review Period
     - 2020-04-17 - 2020-05-04

.. contents::
   :local:
   :depth: 3


Abstract
********

Wherever possible, the edX organization will provide and manage Docker images as the medium for packaging operating dependencies of an Open edX independently deployable application (IDA) rather than providing Ansible playbooks.

The configuration for running IDAs in different environments will be done through a config file per unique [application, environment] combination. How those files are generated and managed is the responsibility of each operator.

All standard operations for running and managing any Open edX IDA will be documented. How those procedures are executed is the responsibility of each operator.

Wherever possible, operating concerns will live within the codebase they are concerned with. Thus it should be expected that all IDAs will provide the following:

* A Dockerfile from which the container image can be built.
* Documented settings files distinguishing required values without defaults (like secrets) from other settings which will defaults reasonable for most production installations.
* An operations manual documenting standard operations required to run and maintain the IDA.
* A changelog to document differences in settings, operating concerns, and application functionality from release to release.

As code is moved to this new paradigm it is important to note that we strive to maintain or improve upon the state of operations without sacrificing features or visibility.  If operations code is removed from one place it must either have a home in some other public place better aligned with this OEP (EG package installation in Dockerfiles) or no longer be needed at all in this model (EG `conditional ansible commands`_ for installing packages on particular operating
systems).

.. _conditional ansible commands: https://github.com/openedx/configuration/blob/ada15bec6e0269042380e340e65675640983e6ba/playbooks/roles/newrelic_infrastructure/tasks/main.yml#L44-L52

Context
*******

Back when edx-platform was the only code running, the Ansible playbooks in the configuration repo were manageable, though still not necessarily easy to understand. As edX has invested in microservices, the configuration repo has grown to support the configuration of all of them. As patterns emerged shared concerns were pulled out into unversioned shared roles making it hard to change without impacting all apps or wiring configurable toggles throughout much of the shared space.

The end result is that the configuration repo has grown into a sprawling codebase that attempts to be everything for everyone trying to configure servers to run Open edX installations. There are a wide variety of roles, plays, and utility scripts with branching paths within each to support being executed against different base operating systems and many various configurable operational needs. None of this is helped by the fact that edX.org operates at a scale that most installations do not. With a mix of edX.org specific code, branching paths that are not regularly tested, and the high amount of required understanding around how all the pieces interact to change any of it, this approach to operating is a clear source of friction for developing and running the platform. Nothing proves this point more than the fact that the edx-platform README now explicitly warns users to leave it to the professionals.

    Installing and running an Open edX instance is not simple. We strongly recommend that you use a service provider to run the software for you.

The proposal below outlines how we can create a cleaner, more intuitive interface for operating Open edX and in doing so help both edX.org and the Open edX community at large achieve better outcome at a faster pace. In brief it is:

* Use Dockerfiles to capture system dependencies and operate the resulting images in all environments.
* Simplify and standardize how to configure an IDA.
* Document standard operations.


Containers
==========

**Why Containers**

For managing microservice architectures containers are an industry standard for the following reasons:

* **Increased portability.**  Run on any host OS that supports containers.
* **Greater efficiency.**  Ability to safely run multiple containers on the same hardware allows cost savings through efficient use of resources.
* **Improved security.** Containers improve the default level of security by isolating applications from the host system and from each other.
* **Improved application development.**  Allows developers to run a more production-like environment locally, preventing discrepancies between environments.
* **Speed.** Start, create, replicate or destroy containers in seconds. Increases velocity for development, deployment, and production operations.
* **Operational simplicity/consistency.**  By isolating application processes from the host OS it is simpler to manage and maintain the host OS. Homogeneous administration of heterogeneous components, reduces the range of skill sets required to operate environments.
* **Rich Tooling Ecosystem.**  The rich ecosystem of tools and integrations available make containers an easy and powerful deployment solution.

**Clear System Dependency Documentation**

For operators who don’t want to run in Docker containers the Dockerfile acts as documentation for the system dependencies they will need to recreate. Everything an operator needs to know about how to set up their environment is all in one place rather than distributed across multiple overlapping and overriding files.

In order to function as documentation for operators Dockerfiles will be well-commented, use `native Dockerfile syntax`_ to describe the image, and never require private resources to build. This means:

* No Ansible should be run as part of the image build.
* Similarly, bash scripts should be avoided.
* If scripts must be used due to a limitation of the Dockerfile commands, the scripts must live within the same codebase as the Dockerfile.
* Default ARG and CMD values should meet the needs of most users without modification.
* Private or custom install requirements, patches, ARGs, CMD values, etc. should be included via a separately managed Dockerfile built on top of the Open edX image for that codebase.


.. _native Dockerfile syntax: https://docs.docker.com/engine/reference/builder/

**Docker Images**

edX will provide Docker images for IDAs that captures the latest code on the master branch as well as images representing named releases. edX will not provide these images for named releases prior to the acceptance and implementation of this OEP (Aspen through and including Juniper at time of writing).

Operators will be able to use these provided images as a base for any private or custom images they need to build for their environments.


Configuration
=============

**Django settings**

Having a single artifact that runs with different configurations increases stability by improving development parity with other deployment environments. edX IDAs already support configuration overrides via a yaml file for production environments, but development and test environments tend to configure the IDA using different code paths via a settings/devstack.py or settings/test.py file.

Additionally it is not clear which settings are required to be overridden and which settings have values that may technically work but are inappropriate for production systems. To alleviate these issues edX Django IDAs will adopt the following settings structure:

.. code-block:: text

  settings
  ├── __init__.py
  ├── required.py
  └── defaults.py


*  ``__init__.py`` - Sourcing our config from this file within the settings directory takes advantage of Django defaults and means that settings will be picked up automatically without needing to specify ``--settings`` anywhere. This entry point would import ``required.py``, ``defaults.py``, and the code to override both from a config file.
*  ``required.py`` - all settings which are required to run and do not have a reasonable production-ready default, e.g. LMS_BASE_URL which will be different per environment.
*  ``defaults.py`` - other settings which will have production-ready defaults

The settings defined in ``required.py`` and ``defaults.py`` files are mutually exclusive, representing all IDA specific settings as well as installed library settings whose values either must be provided or whose defaults are not considered production-ready.

``required.py`` variables must be overridden by operators.  The application will check that operators provided these values, and will not start unless they are set. This allows operators to fail fast rather than finding out about an unset value when users exercise those breaking code paths. Application developers are encouraged to keep the list of required settings to a minimum.

This new settings structure obviates the need for any other python files in the settings directory (such as ``devstack.py``, ``test.py``, etc). The values currently set in those files should be moved to a corresponding ``devstack.yml``, ``test.yml``, etc in the same settings directory. This gives developers and operators more consistency across environments since the same code paths are being executed with different values.


**Config file**

IDAs will be configured by a yaml file containing all of the settings variable overrides specified by the operator (including both required settings and secrets as well as default value overrides). The file is made known to the IDA by an environment variable, ``<APPNAME>_CFG_PATH``, with the path to the file. Versions of this config yaml may be provided in the application repo for certain environments such as development and test. However, for all other environments (e.g. production), the file will need to be managed elsewhere.

Since defaults are provided by the IDA, many smaller deployments should not need to do much more than provide the required settings to operate. For development environments the config will likely change the defaults to more development appropriate values, e.g. debug settings, log levels, email settings, etc.

**Config file generation & management**

Due to the varied needs and processes of different operators, how the config files are created, managed, or otherwise end up on the server is up to the operator and will depend greatly on their deployment strategy. With a consistent method for configuring IDAs it will be reasonable to have tooling to assist with migrating between releases, but the implementation of such tooling is outside the scope of this proposal.

**Documentation of settings**

The settings found in both the ``required.py`` and ``defaults.py`` files will be documented to describe what they are and how they should be used. The documentation will consist of `Sphinx autodoc`_ compatible comments before each setting. For reference that is a ``“comment with special formatting (using a #: to start the comment instead of just #)”``. This keeps documentation close to the code as it is being written, while allowing it to be surfaced in generated docs.


.. _Sphinx autodoc: https://www.sphinx-doc.org/en/master/usage/extensions/autodoc.html#directive-autoattribute

Operations Manuals
==================

A clear manual of operations will exist in the form of RST files in an ``operations`` directory within the ``docs`` directory (as per `OEP-19`_) for that IDA. See `this commit`_ for an example provided by the Open edX Build-Test-Release working group. The operations docs will cover common operations such as how to run the IDA for web traffic or as an async worker and how to manage the IDA's underlying database schema. It will also include a list of potential maintenance tasks operators may want to leverage such as clearing sessions or applying security patches. Finally it will include the list of ad-hoc management commands operators can use to help handle edge case or one-time operations.

In the same vein as not dictating how operators create and manage their IDA config files, operators will also be expected to manage how they execute the operations documented in the manual.

.. _OEP-19: https://docs.openedx.org/projects/openedx-proposals/en/latest/oep-0019-bp-developer-documentation.html
.. _this commit: https://github.com/openedx-btr-wg/edx-platform/commit/18effd83f983f497ca0a1535108fa41dc50d06a2#diff-ca02329742db0a77612a18ba1260d178R1-R39


Industry Best Practices
=======================

The recommendations above are heavily inspired by the following resources:

* https://12factor.net/
* https://code.djangoproject.com/wiki/SplitSettings
* https://github.com/openedx-btr-wg/edx-platform/tree/btr-wg/reference-manual
* https://github.com/openfun/richie/blob/master/src/richie/apps/core/defaults.py


Alternatives Considered
***********************

Refactor configuration
======================

Refactoring the configuration repo to reduce the amount of shared code and making it easier to read and understand what is being run when you execute a particular playbook.

This approach was rejected due to the sheer volume of work required to make this change in an environment that is inherently difficult to test. Also, while it would improve ease of use, it would only provide parity in terms of functionality. Moving to containers will also improve ease of use and unlock many potential future enhancements.

Jumping to Kubernetes
=====================

Kubernetes is an open source container orchestration platform pioneered by Google. While it often occupies the same conversation space as containers because it is a powerful way to manage them, it is a huge increase in complexity and expertise required to operate. For most installations Kubernetes is currently too much overhead/learning curve for the value. The edX organization may opt to explore deploying Docker containers this way in the future and would love to collaborate with operators who also decide to use Kubernetes to compare notes.


Configuration via Environment Variables
=======================================

There are many django project which configure their applications by grabbing the settings value from an environment variable otherwise using a default.  While this is technically feasible, the platform relies on setting many complex data structures (lists & dicts) and to do so using ENV VARS would be quite challenging to manage and thus was declined as an option to pursue.


Implementation Strategy
***********************

Discussion of implementation of this OEP will happen in a `separate Pull Request`_ .

.. _separate Pull Request: https://github.com/openedx/openedx-proposals/pull/144

Related Decisions
*****************

The following related decisions modify or enhance this OEP, but have not yet been fully incorporated as updates to this OEP:

.. toctree::
   :caption: OEP-45 Decisions
   :maxdepth: 1
   :glob:

   oep-0045/decisions/*

Change History
**************

2026-07-03
==========

* Addressing typos in the documentation

2024-05-16
==========
* Change status to "Needs Revision"
* `Pull request #586 <https://github.com/openedx/openedx-proposals/pull/586>`_
