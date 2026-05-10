# OCP Platform

This repository contains the declarative configuration for the OpenShift Container Platform. All components are managed as YAML manifests and applied to the cluster via `oc apply` or through GitOps.

## Prerequisites

- Access to an OpenShift cluster (e.g. CRC for local development)
- `oc` CLI installed and logged in as a user with the `cluster-admin` role
- The cluster has access to the Red Hat Operator Catalog (`redhat-operators`)

## Components

### OpenShift GitOps Operator

The [Red Hat OpenShift GitOps Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.10/html/installing_gitops/installing-openshift-gitops) provides a fully managed ArgoCD instance. It serves as the foundation for GitOps-based management of all further platform and workload configurations.

The installation consists of three manifests:

| Manifest | Description |
|---|---|
| `namespace.yaml` | Creates the `openshift-gitops-operator` namespace for the operator |
| `operatorgroup.yaml` | Defines the OperatorGroup that sets the installation scope |
| `subscription.yaml` | Creates the Subscription that installs the operator via OLM from the `redhat-operators` catalog |

#### Installation

Apply the manifests in the correct order:

```bash
oc apply -f components/openshift-gitops-operator/namespace.yaml
oc apply -f components/openshift-gitops-operator/operatorgroup.yaml
oc apply -f components/openshift-gitops-operator/subscription.yaml
```

#### Verifying the installation

Check the operator pods in the operator namespace:

```bash
oc -n openshift-gitops-operator get po
```

Check the ArgoCD pods in the GitOps namespace — the operator automatically creates a ready-to-use ArgoCD instance here:

```bash
oc -n openshift-gitops get po
```

Once all pods are in `Running` state, the ArgoCD UI is accessible at:

```
https://openshift-gitops-server-openshift-gitops.apps-crc.testing
```

> **Note:** The initial ArgoCD admin password can be retrieved with the following command:
>
> ```bash
> oc -n openshift-gitops extract secret/openshift-gitops-cluster --to=-
> ```

## Directory structure

```
ocp-platform/
├── components/
│   └── openshift-gitops-operator/
│       ├── namespace.yaml
│       ├── operatorgroup.yaml
│       └── subscription.yaml
└── README.md
```
