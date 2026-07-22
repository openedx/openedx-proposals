Provisioning Open edX Infrastructure on Kubernetes
##################################################


Status
******

Accepted


Context
*******

For most of the history of the Open edX project, production deployments were done primarily using Virtual Machines that had been provisioned using the ansible scripts in the ``edx/configuration`` repository. With the adoption of the `decision 0001`_ from OEP-0045 that selects Tutor as the replacement for ``edx/configuration`` some large instances and operators are encountering use cases that are not supported. Multiple instances of Open edX sharing some infrastructure is the first of such cases.

A popular way to deploy containerized applications is to use `Kubernetes`_. However, Kubernetes has a lot of complexity so the overhead it brings is usually not worthwhile for small, singe-instance deployments. For that reason (as explained in OEP-45) as well as a general lack of experience with Kubernetes, to date there has been no officially recommended way to deploy Open edX on Kubernetes. Although Tutor does include basic support for Kubernetes, it is focused only on a basic working deployment, and lacks the flexibility and capability that operators require for large, multi-instance deployments.

Despite this, many major commercial Open edX hosting providers have migrated toward deploying Open edX on Kubernetes. In doing so, they developed several different approaches for doing so, each fairly different from each other but all focused on deploying Tutor-generated images into a Kubernetes cluster. This has resulted in duplicated effort and made it difficult to share best practices and lessons learned with each other. For further context, read the forum discussion `Deploying multiple Open edX instances onto a Kubernetes Cluster with Tutor`_.


.. _Kubernetes: https://kubernetes.io/
.. _Deploying multiple Open edX instances onto a Kubernetes Cluster with Tutor: https://discuss.openedx.org/t/tech-talk-demo-deploying-multiple-open-edx-instances-onto-a-kubernetes-cluster-with-tutor/4641
.. _decision 0001: https://docs.openedx.org/projects/openedx-proposals/en/latest/architectural-decisions/oep-0045/decisions/0001-tutor-as-replacement-for-edx-configuration.html


Decision
********

For Open edX operators who wish to deploy Open edX on Kubernetes, community-maintained `Terraform`_ modules and `Helm Charts`_ will be used to simplify the process of provisioning a Kubernetes cluster capable of loading one or more Open edX instances.

Building on top of the `decision 0001`_, individual instances will be configured by Tutor and the Tutor plugin ecosystem.

As the needs and deployment scale of Open edX operators can vary significantly, the Terraform modules and Helm charts will be designed to be flexible, incorporating sensible defaults but allowing customization of almost every aspect of the deployment. This includes supporting various cloud providers and different underlying technologies for Kubernetes resources. E.g. Ingress controllers.

The goal is for these Terraform modules and Helm charts to be developed and maintained collaboratively, by the Open edX Operators that are using them. All relevant best practices from Open edX, popular Terraform modules, and Helm projects should be followed from the start, such as `OEP-55 Project Maintainers`_.


.. _OEP-55 Project Maintainers: https://docs.openedx.org/projects/openedx-proposals/en/latest/processes/oep-0055-proc-project-maintainers.html
.. _Terraform: https://developer.hashicorp.com/terraform
.. _Helm Charts: https://helm.sh/


Consequences
************

Terraform Modules
=================

Although several IaC solutions are available, Terraform is the most stable and widely recognized option. Its extensive module registry provides APIs and integrations for most cloud providers, SaaS platforms, and even local utilities. Terraform makes it predictable and convenient to set up infrastructure that Helm can then deploy to and operate on.

Terraform modules are written in HCL (HashiCorp Configuration Language), which makes them easy to understand. However, Terraform also adds another layer of complexity to the already complex Open edX stack.


Helm Considerations
===================

Helm is the best known package manager for Kubernetes. It's status as a graduated project from the Cloud Native Computing Foundation is a clear indication of the thriving adoption, open governance and commitment to the open community.

Helm makes it possible to package and publish charts, which can be rendered as kubernetes manifests. These charts provide robust support for versioning, deployment and rollback. At the same time Helm makes it easy for operators to customize application configurations during deployment and for developers to provide sensible defaults.

Charts are built with the Go language templating engine. This is a drawback, since it means adding a new language to the ecosystem with a new learning curve. Nevertheless, this is considered acceptable due to the large adoption of Helm in the devops community.


Best Practices
==============

**Cloud Providers**

As much as possible, the Terraform modules and Helm chart should aim to be agnostic toward the underlying cloud provider used (e.g. AWS, Azure, Google Cloud Platform, etc.). In cases where provider-specific code is required, the Charts should be written in such a way that they can be configured to work with multiple providers, and not e.g. only support AWS.

**Terraform Modularity**

The community-maintained Terraform modules should aim for being standalone and provider agnostic as much as possible. In such cases when it is not possible (e.g., setting up an EKS cluster on AWS), the Terraform module boundaries should be clear enough that they can be used independently. For example, an operator may reuse the community-maintained Terraform module for setting up an EKS cluster on AWS, but that should not pull in dependencies on the RDS cluster module.

**Helm Subcharts and Modularity**

The project should aim to make the charts available in a way that is composable and granular. Extending a single subchart and using it in the composition should be a better and faster alternative than forking the whole project.


Industry Best Practices
=======================

The recommendations above are heavily inspired by the following resources:

* https://helm.sh/docs/chart_best_practices/


Alternatives Considered
***********************

Building more Kubernetes support into Tutor
===========================================

Enhancing the current support for tutor core was not considered since it would go against one of the goals of the project as `explained by the author`_.

.. _explained by the author: https://github.com/overhangio/tutor/pull/675#issuecomment-1140919654

Custom Templating tool
======================

A `custom templating tool`_ built as a tutor plugin and leveraging kustomize for extension was demonstrated and considered as a possible alternative. It was ultimately rejected as Helm would provide more flexibility in extension without the need of maintenance.

.. _custom templating tool: https://github.com/eduNEXT/drydock


Implementation Strategy
***********************

Once this ADR has reached Provisional status:

1. A new repository will be created on GitHub, e.g. at https://github.com/openedx/openedx-helm-charts
2. The authors of this ADR will create a roadmap using GitHub Issues by defining the initial development issues.
3. Invite interested parties to become core contributors on the repo, and nominate some of them.
4. Participating core contributors will then complete the roadmap and begin developing an MVP.


References
**********

A `working proof of concept`_ that was written as part of the research for this ADR. See `2022-11-22 meeting recap`_ for further information.

.. _`working proof of concept`: https://github.com/open-craft/tutor-contrib-multi
.. _`2022-11-22 meeting recap`: https://discuss.openedx.org/t/deploying-open-edx-on-kubernetes-using-helm/8771


Change History
**************

2026-07-03
==========

* Extend the decision with Terraform which is integral part of the repository.
* Rename the decision to reflect its purpose better.

2023-01-30
==========

* Document updated to include community feedback.


2022-11-30
==========

* Document updated to become an ADR of OEP-45 instead of a standalone OEP.


2022-08-19
==========

* Document created
* `Pull request #372 <https://github.com/openedx/openedx-proposals/pull/372>`_
