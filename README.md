# Crossplane - Kubernetes Provider - ProviderConfigRequest - Proof of Concept

This repository contains a proof of concept for the Crossplane Kubernetes Provider feature request [#89](https://github.com/crossplane-contrib/provider-kubernetes/issues/89).

**TL;DR:** The objective is to provide a way to enable teams to submit/configure their own `ProviderConfig`s for the Crossplane Kubernetes Provider.

## Configuration options

There are a few ways to configure a `ProviderConfig`:

### `in-cluster` configuration

```yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: kubernetes-provider
spec:
  credentials:
    source: InjectedIdentity
```

### `kubeconfig` in a secret configuration

```yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: kubernetes-provider
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: cluster-config
      key: kubeconfig
```

### `gcloud` identity configuration

```yaml
apiVersion: v1
kind: Secret
metadata:
 namespace: crossplane-system
 name: example-provider-secret
type: Opaque
data:
  credentials: BASE64ENCODED_PROVIDER_CREDS
---
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: kubernetes-provider
spec:
identity:
  type: GoogleApplicationCredentials
  source: Secret
  secretRef:
    name: gcp-credentials
    namespace: crossplane-system
    key: credentials.json
```

## Use Case

In a scenario where users don't have access to cluster-scoped resources, they can't create their own `ProviderConfig`s.
In addition, providing a `kubeconfig` in a secret is not straightforward for users.

This is why the multi-tenancy experience is not great/enough for the Kubernetes Provider.

## Proof of Concept

This proof of concept configures the following scenario:

- A vanilla Kubernetes cluster.
- Crossplane.
- Kubernetes Provider.
  - A `ProviderConfig` that uses the `InjectedIdentity` source.
  - A `ControllerConfig` using a `ServiceAccount` to authenticate to the Kubernetes cluster.
    - This `ServiceAccount` shouldn't be cluster-admin. **It is in this proof of concept for simplicity.**
- An application namespace.
  - It has a `ServiceAccount` with a few permissions.
    - This is the `ServiceAccount` that will be used by the Kubernetes Provider to authenticate to the Kubernetes cluster.
    - The user should have restricted access to the cluster and the namespace.
      - One of the permissions is to create `ProviderConfigRequests`.

## Install proof of concept

### Prerequisites

```bash
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
# Install Crossplane
$ kubectl create namespace crossplane-system
namespace/crossplane-system created
$ helm repo add crossplane-stable https://charts.crossplane.io/stable
$ helm repo update
$ helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
NAME: crossplane
LAST DEPLOYED: Sat Jan  7 18:11:46 2023
NAMESPACE: crossplane-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Release: crossplane

Chart Name: crossplane
Chart Description: Crossplane is an open source Kubernetes add-on that enables platform teams to assemble infrastructure from multiple vendors, and expose higher level self-service APIs for application teams to consume.
Chart Version: 1.10.1
Chart Application Version: 1.10.1

Kube Version: v1.25.3
```

At this point, you will have a vanilla Kubernetes cluster with Crossplane installed.

### Install the crossplane Kubernetes provider

```bash
$ kubectl apply -f default-kubernetes-provider.yaml
serviceaccount/provider-kubernetes created
clusterrolebinding.rbac.authorization.k8s.io/provider-kubernetes created
provider.pkg.crossplane.io/provider-kubernetes created
controllerconfig.pkg.crossplane.io/provider-kubernetes created
```

This creates:
- A `ServiceAccount` in the `crossplane-system` namespace.
- A `ClusterRoleBinding` that binds the `ClusterRole` `cluster-admin` to the `ServiceAccount`.
- The `Provider` resource.
- The `ControllerConfig` resource that uses the `ServiceAccount` to authenticate to the Kubernetes cluster.

### Configure the Kubernetes Provider

```bash
$ kubectl apply -f kubernetes-provider-config.yaml
providerconfig.kubernetes.crossplane.io/kubernetes-provider created
```

This creates a `ProviderConfig` that uses the `InjectedIdentity` source *(the `ServiceAccount`)*.

### Install the ProviderConfigRequest XRD

```bash
$ kubectl apply -f provider-config-request.yaml
compositeresourcedefinition.apiextensions.crossplane.io/xproviderconfigrequests.x.k8spin.cloud created
composition.apiextensions.crossplane.io/defaultproviderconfigrequest created
```

- This creates a `CompositeResourceDefinition` that defines the `ProviderConfigRequest` resource.
- It also creates a `Composition` that implements the `ProviderConfigRequest` definition.

### Test the ProviderConfigRequest

```bash
$ kubectl apply -f test/init.yaml
clusterrole.rbac.authorization.k8s.io/app-role created
namespace/app-1 created
configmap/app-config created
serviceaccount/app-account created
rolebinding.rbac.authorization.k8s.io/app-role-binding created
```

This creates:
- A `ClusterRole` that allows the `ServiceAccount` to interact with `configmaps`.
- A namespace called `app-1`.
- A `ConfigMap` in the `app-1` namespace.
- A `ServiceAccount` in the `app-1` namespace. This is the `ServiceAccount` that will be used by the Kubernetes Provider to authenticate to the Kubernetes cluster.
- A `RoleBinding` that binds the `ClusterRole` to the `ServiceAccount`.

Then, as a `app-1` user, you can create a [`ProviderConfigRequest`](test/providerConfigRequest.yaml):

```yaml
kind: ProviderConfigRequest
apiVersion: x.k8spin.cloud/v1alpha1
metadata:
  name: kubernetes-provider-config
  namespace: app-1
spec:
  providerConfigRef:
    name: kubernetes-provider # this one has to be injected by a cluster-admin (or a mutation webhook)
  serviceAccountName: app-account
```

```bash
$ kubectl apply -f test/providerConfigRequest.yaml
# After a while, you will see the following:
$ kubectl get providerconfigrequest -n app-1 -o json kubernetes-provider-config | jq -r .status.share.providerConfigRef.name
app-1-app-account-providerconfig
$ kubectl get app-1-app-account-providerconfig -o yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: '{"apiVersion":"kubernetes.crossplane.io/v1alpha1","kind":"ProviderConfig","metadata":{"name":"app-1-app-account-providerconfig"},"spec":{"credentials":{"secretRef":{"key":"kubeconfig","name":"app-account-kubeconfig","namespace":"app-1"},"source":"Secret"}}}'
  creationTimestamp: "2023-01-07T17:20:14Z"
  finalizers:
  - in-use.crossplane.io
  generation: 1
  name: app-1-app-account-providerconfig
  resourceVersion: "2063"
  uid: 291b85ac-7fad-446c-90e8-5b8fb7ef67c1
spec:
  credentials:
    secretRef:
      key: kubeconfig
      name: app-account-kubeconfig
      namespace: app-1
    source: Secret
status: {}
```

This creates a `ProviderConfig` that uses the `app-account` `ServiceAccount` in the `app-1` namespace to authenticate to the Kubernetes cluster.

#### Bonus: Test the generated `kubeconfig`

The `ProviderConfigRequest` generates a `Secret` with the `kubeconfig` that is used by the `ProviderConfig` to authenticate to the Kubernetes cluster.
But that `kubeconfig` could be used by other tools like `kubectl` to interact with the Kubernetes cluster.

```bash
$ kubectl apply -f test/test.yaml
job.batch/list-configmaps created
$ kubectl logs jobs/list-configmaps -n app-1
NAME               DATA   AGE
app-config         1      8m45s
kube-root-ca.crt   1      8m45s
```

### Considerations

A policy engine like OPA/Gatekeeper would be a great addition to this proof of concept.

- The `ProviderConfigRequest` needs to set the `spec.providerConfigRef` to a `ProviderConfig` that is allowed by the policy engine.
- Then, `Object` resources needs to set the `spec.providerConfigRef` to the namespaced `ProviderConfig` that is allowed by the policy engine in the specific namespace.
- The content of this repository is just a proof of concept. **It is not production-ready**

### Achievements

- The `ProviderConfigRequest` creates a `ProviderConfig` from a `ServiceAccount` in a namespace.

# LICENSE

Read the [LICENSE](LICENSE) file for details.
