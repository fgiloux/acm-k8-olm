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

## Compliance operator

The current version of the compliance operator does not work with the enforcement of PodSecurity to `restricted`.
This leads to the following error message when the label is set accordingly to the namespace:
~~~
  Warning  FailedCreate  84s               replicaset-controller  Error creating: pods "compliance-operator-c68c68d9d-8q8j2" is forbidden: violates PodSecurity "restricted:latest": unrestricted capabilities (container "compliance-operator" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "compliance-operator" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "compliance-operator" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
~~~

The compliance operator relies on a [deprecated label that has been removed in Kubernetes 1.24](https://kubernetes.io/blog/2022/04/07/upcoming-changes-in-kubernetes-1-24/#api-removals-deprecations-and-other-changes-for-kubernetes-1-24): `node-role.kubernetes.io/master` but is still present in OpenShift 4.11.
The new label is `node-role.kubernetes.io/control-plane`.
This requires the addition of the label to the cluster nodes for newly created non-OpenShift distributions.

The compliance operator relies on the generation of a serving certificate, which is an OpenShift specific mechanism. Such a certificate is generated and stored in a secret when an annotation is added to a service:
~~~
$ kubectl get svc metrics -n compliance-operator -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: compliance-operator-serving-cert
~~~

The mount of the secret is marked as optional in the operator Deployment and does not prevent the Pod to get started:
~~~
 Volumes:
  serving-cert:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  compliance-operator-serving-cert
    Optional:    true
~~~

But the operator process fails later on:
~~~
{"level":"error","ts":"2023-02-03T15:40:30.767Z","logger":"setup","msg":"Error creating metrics service/secret","error":"compliance-operator-serving-cert not found - restarting, as the service may have just been created","stacktrace":"github.com/ComplianceAsCode/compliance-operator/cmd/manager.addMetrics\n\tgithub.com/ComplianceAsCode/compliance-operator/cmd/manager/operator.go:337\ngithub.com/ComplianceAsCode/compliance-operator/cmd/manager.RunOperator\n\tgithub.com/ComplianceAsCode/compliance-operator/cmd/manager/operator.go:294\ngithub.com/spf13/cobra.(*Command).execute\n\tgithub.com/spf13/cobra@v1.5.0/command.go:876\ngithub.com/spf13/cobra.(*Command).ExecuteC\n\tgithub.com/spf13/cobra@v1.5.0/command.go:990\ngithub.com/spf13/cobra.(*Command).Execute\n\tgithub.com/spf13/cobra@v1.5.0/command.go:918\nmain.main\n\t./main.go:45\nruntime.main\n\truntime/proc.go:255"}
~~~


