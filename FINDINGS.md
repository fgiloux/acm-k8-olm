# Findings

This page captures the findings in terms of gaps and pitfalls impacting the user experience.

## Registry credentials and mirroring

In OpenShift multiple operator catalogs are available. The images for the Red Hat container catalog are stored in Red Hat registry, which requires authentication. Besides the subscription aspect to make these images available in a Kubernetes cluster the container-engines on the nodes need to be configured with the required credentials and CA certificate. The alternative of providing the credentials to a specific ServiceAccount, as it has been done for the deployment of the ACM components is not viable, when every operators and their operands are run with a different ServiceAccount.

In a disconnected environment there is in addition the need to mirror external registries, like the Red Hat's one. This again requires a change in configuration of the container-engines.

These configurations can be centrally defined in a custom resource and applied to the nodes of an OpenShift cluster thanks to the Machine Config Operator (MCO). This, however, is not available with a different Kubernetes distribution, although similar mechanisms may be available by cloud providers.

Here a few options that may be considered:
- to use ACM integration with Ansible to automate the configuration of the hosts
- to have the container engine configuration backed in the operating system image, e.g. a custom AWS AMI
- to use a higher level API, possibly interacting with it through a DaemonSet [like in AKS](https://github.com/Azure/AKS/issues/1940#issuecomment-995894445)