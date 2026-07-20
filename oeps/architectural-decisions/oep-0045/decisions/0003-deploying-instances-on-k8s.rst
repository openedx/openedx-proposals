Deploying Open edX on Instances Kubernetes
##########################################

Status
******

Draft


Context
*******

Hosting Open edX instances on Kubernetes requires a complex infrastructure setup, especially when multiple instances are involved. Decisions made during cluster setup can directly affect the performance, latency, and reliability of those instances

The `decision 0002`_ elaborates on the need for community-maintained components to reduce this complexity. Although the Terraform modules and Helm chart support a modular setup already removing a significant part of the complexity, they do not fully abstract that away. Furthermore, none of these components are addressing the significant challenge of deploying instances on Kubernetes.

Open edX commercial providers are addressing this challenge with purpose-made solutions that are mostly not compatible with each other, although they are built using some community-maintained and provided components.

.. _decision 0002: https://docs.openedx.org/projects/openedx-proposals/en/latest/architectural-decisions/oep-0045/decisions/0002-openedx-hosting-infrastructure-on-k8s.html


Decision
********

We recommend that all large Open edX deployments use the community-maintained Terraform modules and Helm chart described in decision 0002, existing solutions in the Open edX ecosystem (`GitHub Actions`_), provider-maintained open-source tooling (`Drydock`_ and `Picasso`_), and cloud-native, provider-agnostic deployment tools (`ArgoCD`_, and `Argo Workflows`_). We encourage providers to share any additional deployment code/tooling they use as open source repos or templates that other providers can use as a reference. If any particular template such as OpenCraft's Launchpad (link) becomes adopted by multiple providers, it should be considered for adoption as a community standard, and this OEP should be updated to recommend its use.

.. _GitHub Actions: https://github.com/features/actions
.. _Drydock: https://github.com/eduNEXT/drydock
.. _Picasso: https://github.com/eduNEXT/picasso
.. _ArgoCD: https://argo-cd.readthedocs.io/en/stable/
.. _Argo Workflows: https://argoproj.github.io/workflows/


Consequences
************

Build Pipeline Considerations
=============================

In order to build Open edX instance images with Tutor, the CI/CD pipelines should be executed. The cluster template is going to use Picasso to simplify the image building process, though this binds the rendered clusters to GitHub Actions.


Deployment Pipeline Considerations
==================================

The deployment of instances are going to be handled by ArgoCD that reads the Kubernetes manifests rendered by Tutor and pushed to the cluster repository by GitHub Actions. Although this keeps continuous deployment convenient, easy to oversee, and ensures that every change can be audited, secrets are ending up in the version control system.

This is the result of some limitations coming from Tutor-rendered Kubernetes manifests.


Best Practices
==============

In order to keep a healthy state for the cluster template, these best practices should be followed:

* All infrastructure dependency should be coming from the community-maintained and provided Terraform modules and Helm charts, laid out in decision 0002.
* Per-instance resource provisioning (e.g., database users) should be as much provider-agnostic as possible.
* No company- or Open edX provider-specific logic should live in the cluster template or rendered cluster.
* The instance deployment should be handled by cloud provider-agnostic tooling.
* The CI/CD pipelines should do only the bare-minimum needed for instance setup.
* No fragile or dangerous operations should be performed by the CI/CD pipelines except the instance deprovisioning flow.
* The operators should handle the infrastructure modifications from their computer rather than from CI/CD pipelines.
* All infrastructure provisioning should be done by purpose-made native and cluster template provided CLI tooling described by the cluster template repository.


Change History
**************

2026-07-03
==========

* Document created
