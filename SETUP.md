# Setup

These are the instructions for installing and configuring OLM on Kubernetes clusters through ACM.

## Local environment

The instructions in this page assume that you already have available ACM and a Kubernetes cluster to be managed by ACM.
For setting up a local test environment please confer to [LOCAL.md](./LOCAL.md)

## Cluster registration

### Hub cluster preparation

Log into the hub cluster.
~~~
$ oc login
~~~

Create a new namespace.
~~~
$ oc new-project kind-1
~~~

Create a managed cluster resource.
~~~
$ cat <<EOF | oc apply -f -
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: kind-1
  labels:
    cloud: auto-detect
    vendor: auto-detect
spec:
  hubAcceptsClient: true
EOF
~~~

### Managed Kubernetes cluster preparation

Create a namespace and a ServiceAccount for ACM.
~~~
$ kubectl create ns acm
$ kubectl create sa acm -n acm
~~~

Give enough rights to the acm service account for managing the Kubernetes cluster (cluster-admin role for the purpose of the PoC).
~~~
$ cat <<EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: acm-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: acm
  namespace: acm
EOF
~~~

Create a ServiceAccount token (auto-generated) for the previously created ServiceAccount.
~~~
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: acm-token
  namespace: acm
  annotations:
    kubernetes.io/service-account.name: acm
type: kubernetes.io/service-account-token
EOF
~~~

Retrieve the URL of the API of the Kubernetes cluster. This is for instance available in its kubeconfig.
 ~~~
 $ kubectl config view | grep server
 ~~~
 This gives `https://192.168.130.1:37015` with my local setup.

 Get the generated token from the ServiceAccount secret with type kubernetes.io/service-account-token created in the previous step.
 ~~~
 $ export TOKEN=$(kubectl get secret acm-token -n acm -o jsonpath="{.data.token}" | base64 -d)
 ~~~

### Provide credentials to ACM

Create the secret with the token on the hub cluster.
~~~
$ cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: auto-import-secret
  namespace: kind-1
stringData:
  autoImportRetry: "5"
  token: $TOKEN
  server: https://192.168.130.1:37015
type: Opaque
EOF
~~~
- *`$TOKEN` is the token retrieved in a previous step.*
- *`https://192.168.130.1:37015` is the API URL retrieved in a previous step.*

## OLM installation

### Approach

With a policy it is possible to enforce the creation of resources that match the manifests required for installing OLM, i.e. for release v0.23.1:
- [CustomResourceDefinitions](https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.23.1/crds.yaml)
- [Resources](https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.23.1/olm.yaml)

### Policy generation

The [policy generator plugin](https://github.com/stolostron/policy-generator-plugin) allows to easily create policies from a set of manifest. If not already available on your system follow the instructions to install kustomize and the plugin.

This repository contains [OLM v0.23.1 manifests](./policies/olm/manifests), the required [configuration of the policy plugin generator](./policies/olm/policyGenerator.yaml) and a [PlacementRule](./policies/olm/rules/placementrule.yaml) for the generation of the desired policy.

In the root directory of this repository it can be done with the following command.
~~~
$ kustomize build policies/olm/ --enable-alpha-plugins > policy-olm.yaml
~~~

### Policy deployment

If it is not already done activate the add-ons for the governance-policy-framework on the hub cluster.
~~~
$ cat <<EOF | oc apply -f -
apiVersion: agent.open-cluster-management.io/v1
kind: KlusterletAddonConfig
metadata:
  name: kind-1
  namespace: kind-1
spec:
  applicationManager:
    enabled: true
  certPolicyController:
    enabled: true
  iamPolicyController:
    enabled: false
  policyController:
    enabled: true
  searchCollector:
    enabled: true
EOF
~~~

When the add-ons have been installed the new policy can get applied to ACM hub cluster.
~~~
$ oc new-project k8-policies
$ oc create -f policy-olm.yaml
~~~

The PlacementRule in the policy ensures that the manifests get copied from the hub cluster to the newly registered Kubernetes managed cluster.

The status of the policy can be checked on the hub cluster and the deployment of the OLM components on the managed cluster in the olm namespace.

### Installation with OLM of an operator on the Kubernetes cluster

An operator can now get installed by creating an OLM subscription directly or through ACM on the managed Kubernetes cluster.

Using KEDA as an example.

~~~
$ kubectl create -f https://operatorhub.io/install/keda.yaml
subscription.operators.coreos.com/my-keda created
$ kubectl -n operators get csv
NAME          DISPLAY   VERSION   REPLACES      PHASE
keda.v2.8.1   KEDA      2.8.1     keda.v2.7.1   Succeeded
~~~