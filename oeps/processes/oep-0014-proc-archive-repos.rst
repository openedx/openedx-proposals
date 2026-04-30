.. _OEP-14:
.. _OEP-14 Archiving Open edX GitHub Repositories:

OEP-14: Archiving Open edX GitHub Repositories
###############################################

+---------------+---------------------------------------------------------------+
| OEP           | :ref:`OEP-14 <OEP-14 Archiving Open edX GitHub Repositories>` |
+---------------+---------------------------------------------------------------+
| Title         | Archiving edX GitHub Repositories                             |
+---------------+---------------------------------------------------------------+
| Last Modified | 2022-06-21                                                    |
+---------------+---------------------------------------------------------------+
| Author        | Christina Roberts <christina@edx.org>                         |
|               | Feanil Patel <feanil@axim.org>                                |
+---------------+---------------------------------------------------------------+
| Arbiter       | Nimisha Asthagiri <nasthagiri@edx.org>                        |
+---------------+---------------------------------------------------------------+
| Status        | Accepted                                                      |
+---------------+---------------------------------------------------------------+
| Type          | Process                                                       |
+---------------+---------------------------------------------------------------+
| Created       | 2017-01-09                                                    |
+---------------+---------------------------------------------------------------+
| `References`  | `ORA PR Discussion`_,                                         |
|               | `Moving More Repos to edx-unsupported`_                       |
+---------------+---------------------------------------------------------------+

.. _ORA PR Discussion: https://github.com/openedx/edx-ora/pull/187
.. _Moving More Repos to edx-unsupported: https://openedx.atlassian.net/browse/ARCHBOM-1481

Abstract
********

The `openedx organization`_ contains a large number of repositories, most of which are active and maintained, but some of which are now obsolete. To clarify the status of repositories, a process for archiving a repository is defined below.

.. _openedx organization: https://github.com/openedx

Motivation
**********

Recently openedx.yaml files were added to edX repositories per :ref:`OEP-2 <OEP-2 Repository Metadata>`. In the course of deciding owners for those repositories, there was an `ORA PR Discussion`_ about how best to handle deprecated or obsolete repositories. In particular, do obsolete repositories need owners, and how can repositories be clearly marked as present for archive purposes only?

This discussion resurfaced related to edX's usage of Gemnasium to report the usage of third-party libraries that have known security issues. All repositories under the Open edX organization were being monitored, but this added noise when trying to understand the number of third-party library updates required for actively maintained repositories.


Specification
*************

When a repository under the `openedx organization`_ will no longer be maintained by the Open edX community because it is no longer in use by the latest version of the Open edX platform, the following steps should be followed.

This process is not for repositories that are currently still in use by either the latest release or the master branches of the Open edX platform.  If a repository is in use, but planned to be removed, it should go through the `deprecation process`_ and when that is completed it can be archived as described by the process below.

.. _deprecation process: https://docs.openedx.org/projects/openedx-proposals/en/latest/oep-0021-proc-deprecation.html


Transfer to New Owner if Interest
=================================

First, if the repository is public, and a part of Open edX releases, follow these steps to see if anyone would like to take ownership of it:

1. Post a notice to `Open edX Deprecation Announcements`_ announcing that the repository will be archived, and inquiring if anyone would like to take ownership of the repo. Cross post to the ``#general`` and ``#openedx-proposals`` channels in the `Open edX slack`_. If there are no responses after 2 work days, skip to `Archive Steps`_.

2. If someone does wish to take ownership of the repository, post a new notice to `Open edX Deprecation Announcements`_, clearly indicating who the new proposed owner is, how much time they have to spend maintaining the repo, and when the transfer will take place. Cross post in the above mentioned Slack channels. Wait at least 2 business days before proceeding.

3. Create a new `GitHub Request on the Axim board`_ for the repository to be transferred to the new organization.

4. Once the transfer has occurred, create a fork of the transferred repository into the `openedx organization`_ and follow the `Archive Steps`_ below for the forked repo.

.. _Open edX Deprecation Announcements: https://discuss.openedx.org/c/announcements/deprecation
.. _Open edX slack: http://openedx.org/slack
.. _GitHub Request on the Axim board: https://github.com/openedx/axim-engineering/issues/new/choose


Archive Steps
=============

These steps should be followed for all repos within the Open edX organization (forks included). After some experiments with keeping archived repos in the `openedx github organization`_, we've learned that having abandoned code show up in searches hinders work to understand the current state of the system and the risk around new work, particularly deprecations and API changes. Thus we decided to move all archived repositories to a separate org.

#. Update the README.rst file in the repository to add a brief note about why the repo is being archived, and what is serving as its replacement (where applicable). This may be as simple as a linking to the appropriate DEPR ticket.

#. Close all open PRs and Issues.

#. Unless you have the relevant permissions to perform this step, create a new `GitHub Request on the Axim board`_  and ask them to do the following:

  - Transfer the associated DEPR issue from the to-be-archived repo to `public-engineering`_ (if applicable).

    - Note: the actual Github transfer request takes a little time, so be patient to avoid creating multiple tickets.

  - Archive the repository per `GitHub's archive process`_

  - Move the repository to the openedx-unsupported organization

  - Remove references to the repository from the ``openedx-translations`` `repository <https://github.com/openedx/openedx-translations/>`_

    - The entry in the ``extract-translation-source-files`` `workflow <https://github.com/openedx/openedx-translations/blob/main/.github/workflows/extract-translation-source-files.yml>`_

    - The subdirectory in the ``translations`` `directory <https://github.com/openedx/openedx-translations/tree/main/translations>`_

  - Create an ``axim-engineering`` `request <https://github.com/openedx/axim-engineering/issues/new/choose>`_ to remove the repository resource from the ``openedx-translations`` project on Transifex

.. note::
    Over the lifetime of Open edX, we may fork the same external open source repository multiple times.  In this case, we may need to archive the fork multiple times as we move between our fork and following upstream.  When this is necessary, if possible un-archive the old fork and update it.  If you've already made a new fork, delete the old copy of the fork before you move the new repo to openedx-unsupported.

    This may break some older version of Open edX.  Some combination of copying branches between forks, renaming branches, and changing unsupported versions of Open edX would have to be done to keep things working.  We opt not to take on this extra work by default though may do so under extenuating circumstances.

    One such circumstance is if the previous fork is being used by a supported Open edX named release. In this case, one option would be to port any referenced branches in the old fork to the new fork before deleting the old fork.

.. _openedx github organization: https://github.com/openedx
.. _public-engineering: https://github.com/openedx/public-engineering/
.. _GitHub's archive process: https://help.github.com/en/articles/archiving-repositories

Rationale
*********

We previously archived in place and move to this previously rejected alternative based on lessons learned in going through the deprecation process and major upgrades (Python 3, Django 2.x)

Pros:

- `openedx organization`_ is no longer littered with unsupported/obsolete repositories.
- GitHub search results within the `openedx organization`_ do not include matches in archived repositories. This could decrease confusion, especially since repo descriptions are not included in search results.
- Gemnasium monitoring may cease automatically (although this would need to be confirmed).
- Pattern followed by `Facebook`_, and thus might be familiar to others.

(see `Rejected Alternatives`_ for other options we considered).

.. _Facebook: https://github.com/facebookarchive


Backward Compatibility
**********************

This proposal does not introduce any backward compatibility issues.


Reference Implementation
************************

The `Discussions Hackathon repository`_ has been updated to conform to the `Archive Steps`_.

.. _Discussions Hackathon repository: https://github.com/edx-unsupported/discussions


Rejected Alternatives
*********************

There are a couple variations of this proposal that were originally discussed. Many of the steps of updating documentation and notifying the open source community are essentially the same; the major differences from the proposed process are outlined below.


Alternative 1: Archive In Place
===============================

Use Github's archive feature and updated documentation to archive the repository in place.

Pros:

- Old code hasn't moved so it can be easily found.
- Old code can show up in searches to find historical context.

Reasons rejected:

- Through some experience with this method, we've learned that it's less valuable than we expected.
- Being able to know whether code is alive or dead is really helpful when making major changes and if dead code can't easily be filtered from searches it slows us down.

Alternative 2: Create Archive Branch
====================================

Move the code from the master branch to an archived branch, while leaving the repository itself within `openedx organization`_.

Pros:

- No need to create and maintain a new organization.
- Gemnasium monitoring will cease automatically.
- No help tickets to IT or DevOps are required.
- This pattern was recommended on `Anselm Hannemann's blog`_, though it is not known how many organizations (if any) have adopted this process.

Reasons rejected:

- Non-intuitive, and could be confusing for developers to understand the state of the code, as cloning the repo or viewing it on GitHub would show an empty repository (Note: this could possibly be improved by changing the default branch for the repository, but that might reintroduce the Gemansium monitoring issue).
- It is unclear what the implications would be for any existing forks.

.. _Anselm Hannemann's blog: https://helloanselm.com/2013/handle-deprecated-unmaintained-repositories/


Change History
**************

2026-04-30
===========

* Add step to close open PRs & Issues when archiving a repo.
* `Pull request #786 <https://github.com/openedx/openedx-proposals/pull/786>`_

2025-06-12
===========

* Change title of OEP to "Archiving Open edX GitHub Repositories" from "Archiving edX GitHub Repositories"

2022-06-21
==========

* Fix OEP-14 directory to be "processes" instead of "architectural-decisions".
* Add step for transferring DEPR issue before archiving repo.

2022-09-07
==========

* Reversed order of "Change History".
* Remove example using older edX GitHub orgs to better clarify that this OEP only covers openedx org repos.

2022-01-26
==========

* Change references to ``edx`` GitHub org to ``openedx`` org
* Change internal edX procedures to community-based ones

2022-01-10
==========

* Update instructions to use the openedx-unsupported repo instead of the edx-unsupported repo.

2021-05-27
==========

* Removed step of adding ``[ARCHIVED]`` to the repo name. Github's "archive this repo" setting is now available and is a sufficient indicator.
* Removed step of adding paragraph to README about what archiving means now that we use Github's "archived" marker; the concept of an unmaintained repository and its dangers should be familiar to developers. Keep recommendation to add an explanation of *why* it was archived.
* Removed ``openedx.yaml`` update steps, since the rest of the archive process is sufficient.

2020-09-15
==========

* Updated to provide more details around archiving the same fork multiple times.

2020-03-20
==========

* Decide to use the new edx-unsupported org for all archived repos. Old way we were doing things is now recorded as Alternative 1: Archive In Place.

2019-05-16
==========

* Updated to use GitHub's archive capability.
* Don't ask the community about public repos in the edx org that are not a part of Open edX.

2017-05-23
==========

* Added steps for repositories that live in the edX org, but are forks of other, independent repositories

2017-01-18
==========

* Original publication
