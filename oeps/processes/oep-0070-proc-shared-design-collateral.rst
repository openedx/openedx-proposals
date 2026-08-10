.. _OEP-70 Shared Design Collateral Contribution Requirements:

OEP-70: Shared Design Collateral Contribution Requirements
##########################################################

.. list-table::
   :widths: 25 75

   * - OEP
     - :ref:`OEP-0070 <OEP-70 Shared Design Collateral Contribution Requirements>`
   * - Title
     - Shared Design Collateral Contribution Requirements
   * - Last Modified
     - 2026-07-27
   * - Authors
     - Sam Daitzman <sam.daitzman@schema.education>
   * - Arbiter
     - TBD (Needs an arbiter)
   * - Status
     - Draft
   * - Type
     - Process
   * - Created
     - 2026-07-14
   * - Review Period
     - TBD (recommended 2 weeks)
   * - References
     - | Open edX shared Figma instance
       | `Open edX Shared Design Collateral
       <https://openedx.atlassian.net/wiki/x/A4BtbwE>`_
       |  :ref:OEP-1 OEP Purpose and Guidelines`

.. contents::
   :local:
   :depth: 1

Abstract
********

This OEP establishes that design collateral contributed to the Open edX
platform must be contributed back into the shared Open edX Figma instance,
under open access for reuse, as a condition of the work being considered
complete. This applies to all design contributions to the platform. It applies
the same collaboration model the community already relies on for code to
design: a single source of truth, versioned work in branches, and maintainer
review before merge.

Following the precedent of prior *amnesty*-style process changes (e.g., the
:ref:`lint amnesty <OEP-34 Lint Amnesty>`),
the requirement applies going forward only: existing and legacy design files
are granted amnesty and are not required to be retroactively migrated to match
the structure or format of the standard reference design collateral, though
contributing them is strongly encouraged.

Motivation
**********

Historically, the Open edX project has had no shared source of truth for design resources.
This created three recurring problems:

#. **Duplicated, siloed work.** Every provider with design capacity maintained
   separate, out-of-date versions of designs, and teams regularly recreated
   designs from scratch for existing Open edX apps.
#. **Ambiguous intent.** Without a design source of truth, engineers could not
   differentiate intentional design decisions from provider-specific Figma
   quirks or accidents.
#. **High onboarding cost.** New design contributors had nowhere to start and
   no history of decisions to build on, making community onboarding very
   difficult.

The shared Open edX Figma instance now provides that source of truth, with
resources across nearly all platform areas and modern, reusable file structures
aligned to Paragon. However, **adoption is now the central risk to future Open
edX community shared/open design efforts.** Strong in-person interest has not
translated into active use of the shared instance, and some active
contributions have built on openly shared components without contributing their
new files back or making them available for open reuse.

Without a community-wide norm, a single provider effectively becomes the only
party keeping the files current, which is not sustainable. Codifying a
contribution expectation is the most direct lever to secure the long-term value
of the shared instance.

Specification
*************

#. **Contribution requirement.** Any accepted product proposal that includes
   design work must submit its design collateral to the shared Open edX Figma
   instance for the project to be considered complete. This should be captured
   as an expectation in future design contributions.

   - **Note**: the contribution model is actively maintained by the `Design
     <https://openedx.atlassian.net/wiki/x/KwDR3w>`_ and `Frontend
     <https://openedx.atlassian.net/wiki/x/9YYuu>`_ Working Groups, which will
     allow for flexibility and openness in shared design collateral governance
     going forward.

#. **Open access.** Contributed files must be openly accessible and reusable by
   the community (viewable and exportable, under an open license) so that
   anyone can reference or build on them, including for open-format export and
   agentic prototyping.
#. **Workflow parity with code.** Contributions happen in branches within the
   shared instance. Design maintainers review work-in-progress and provide
   feedback before it is merged into the live product-area files, mirroring the
   code-maintainer model. Frozen release files are published to the Figma
   Community as part of the release (Build-Test-Release, or BTR) process.
#. **Roles.** Each proposal and contribution stream has identified Author(s)
   and an Arbiter, plus Design Maintainers (analogous to code maintainers)
   responsible for reviewing and merging contributions.
#. **Communication and relay.** This expectation is relayed to current and
   future contribution owners and providers, and at minimum current
   contribution owners are asked to return files they worked on to the shared
   instance.

Backward Compatibility
**********************

This OEP applies to new and future work only: **no regressions.** Design work
that predates this proposal is granted amnesty: providers are not required to
retroactively migrate legacy files into the shared instance, though doing so is
strongly encouraged. This mirrors the community's linting-amnesty precedent,
where new violations are disallowed while pre-existing ones are forgiven.

Rationale
*********

- **Proven model.** Code contributors already benefit from a shared source of
  truth, versioning, parallel work, central issue tracking, and a shared doc
  site (Paragon). Extending the same practices to design is a natural, low-risk
  step.
- **Lower barrier to entry.** A shared, openly licensed instance lets new
  contributors start from existing work instead of an empty canvas,
  accelerating onboarding.
- **Cleaner handoffs.** A design source of truth lets engineers distinguish
  intentional decisions from accidents, reducing guesswork and rework.
- **Reduced duplication.** Shared, reusable atoms, molecules, and full views
  cut redundant design spend across providers.

Open Questions
**************

- Exact licensing terms and access mechanics for contributed files.
- Whether the requirement applies to all design contributions or a defined
  subset or tier.
- The support and resourcing model (e.g., a pilot and training) to help
  providers meet the requirement.

Change History
**************

2026-07-14
==========

* Document created.
* `Pull request #813 <https://github.com/openedx/openedx-proposals/pull/813>`_
