# Catalog

These are instructions for creating a custom catalog with a single operator and deploying the catalog and operator to managed Kubernetes clusters through a policy.

## Creation of the catalog index

Prerequisites
- a recent version of opm (tested with the version built on 2023-01-05)
- podman version 1.9.3+

1. Write a [Dockerfile](./catalog/Dockerfile) to build the index image.

2. Copy files providing [an icon](./catalog/content/icon.png) and [a description](./catalog/content/README.md) in a subdirectory.

3. Create a subdirectory `compliance-operator-index` in the `catalog` directory and let generate the catalog index into it:

~~~
$ opm init compliance-operator \
    --default-channel=release-0.1 \
    --description=./content/README.md \
    --icon=./content/icon.png \
    --output yaml \
    > compliance-operator-index/index.yaml
~~~

4. Add the desired compliance operator bundle to the catalog index

~~~
$ opm render registry.redhat.io/compliance/openshift-compliance-operator-bundle@sha256:6dfb7cf56f56e55506a7beed57984d52457666eaa344ebcd860dbc21d13e1d7d \
    --output=yaml \
    >> compliance-operator-index/index.yaml
~~~

5. Add a channel entry to the catalog index

~~~
$ cat >> compliance-operator-index/index.yaml << EOF
---
schema: olm.channel
package: compliance-operator
name: release-0.1
entries:
  - name: compliance-operator.v0.1.59
EOF
~~~

6. Validate the catalog index

~~~
$ opm validate compliance-operator-index
$ echo $?
0
~~~

7. Build the catalog image and push it to the desired container registry

~~~
$ podman build . \
    -f Dockerfile \
    -t quay.io/fgiloux/compliance-operator-index:v0.1.59

$ podman push quay.io/fgiloux/compliance-operator-index:v0.1.59
~~~

The catalog index is now available.

## Policy creation for the deployment

The [policy generator configuration](./policies/policyGenerator.yaml) used during the setup has been extended for generating policies for the deployment of the CatalogSource and the Compliance Operator.
The previously created PlacementRule is reused for the new policies.

### CatalogSource

A [CatalogSource](./policies/catalog-manifests/catalogsource.yaml) pointing to the operator index is available and an entry in the configuration of the policy generator points to it.

~~~
- name: catalog
  disabled: false
  manifests:
    - path: catalog-manifests/
  remediationAction: enforce
  categories:
    - CM Configuration Management
  controls:
    - CM-2 Baseline
  standards:
    - NIST 800-53
~~~

### Compliance Operator

Manifests for a [Namespace](./policies/compliance-manifests/namespace.yaml), an [OperatorGroup](./policies/compliance-manifests/operatorgroup.yaml) and a [Subscription](./policies/compliance-manifests/subscription.yaml) for the installation of the compliance operator are available. An entry in the configuration of the policy generator points to their directory.

~~~
- name: compliance
  disabled: false
  manifests:
    - path: compliance-manifests/
  remediationAction: enforce
  categories:
    - CM Configuration Management
  controls:
    - CM-2 Baseline
  standards:
    - NIST 800-53
~~~

## Policies generation and deployment

The policies were generated and deployed during the setup along the OLM policy.

~~~
$ kustomize build policies/ --enable-alpha-plugins > policies.yaml
$ oc create -f policies.yaml
~~~