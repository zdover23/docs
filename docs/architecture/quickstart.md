---
CobaltCore Quickstart Guide
---

# CobaltCore Quickstart Guide

In this guide we deploy Ceph through Rook, and we install and configure several other components that are necessary to support the CobaltCore project. 

> **Note:** This guide is intended as a starting point and may be incomplete
> in places where CobaltCore's upstream documentation is still being developed.
> Sections marked with **[TODO]** require additional information.

---

## Prerequisites

* Access to a Kubernetes cluster
* `kubectl` properly configured
* Helm


### Storage

Your worker nodes must have raw block devices available for Ceph OSDs
(unformatted, not mounted, not in use by the operating system).

**[TODO]:** Confirm minimum storage requirements per OSD for a CobaltCore
deployment.

---

## Step 1: Set Up the Namespace

Set a namespace variable and create the namespace in your cluster:

```bash
COBALTCORE_NAMESPACE=cobaltcore
kubectl create namespace "$COBALTCORE_NAMESPACE"
```

---

## Step 2: Install Prerequisites

### cert-manager

All internal communication in CobaltCore is encrypted. Install cert-manager
to manage certificates:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.yaml
```

Wait for cert-manager to be ready before proceeding:

```bash
kubectl -n cert-manager rollout status deployment/cert-manager
kubectl -n cert-manager rollout status deployment/cert-manager-webhook
```

### Certificates and Issuers

Generate a root CA and install it as a Kubernetes secret:

```bash
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt -subj "/CN=CobaltCore-CA"
kubectl -n "$COBALTCORE_NAMESPACE" create secret tls root-ca --key ca.key --cert ca.crt
```

---

## Step 3: Install Rook and Deploy a Ceph Cluster

CobaltCore uses [Rook](https://cobaltcore-dev.github.io/docs/architecture/rook.html)
to manage the Ceph storage cluster within Kubernetes.

### Install the Rook Operator

Clone the Rook repository and deploy the operator:

```bash
git clone --single-branch --branch release-1.14 https://github.com/rook/rook.git
cd rook/deploy/examples

kubectl apply -f crds.yaml
kubectl apply -f common.yaml
kubectl apply -f operator.yaml
kubectl apply -f csi-operator.yaml
```

Wait for the operator to be ready:

```bash
kubectl -n rook-ceph rollout status deployment/rook-ceph-operator
```

### Deploy the Ceph Cluster

```bash
kubectl apply -f cluster.yaml
```

Watch the deployment progress:

```bash
kubectl -n rook-ceph get pods -w
```

Wait until all monitor, manager, and OSD pods are in `Running` state before
proceeding.

### Verify Cluster Health

Install the Rook toolbox and check cluster health:

```bash
kubectl apply -f toolbox.yaml
kubectl -n rook-ceph rollout status deployment/rook-ceph-tools
kubectl -n rook-ceph exec -it deployment/rook-ceph-tools -- ceph status
```

A healthy cluster will show `HEALTH_OK`.

---

## Step 4: Deploy the Arbiter Operator

The [Arbiter](https://cobaltcore-dev.github.io/docs/architecture/arbiter.html)
operator deploys external Ceph monitors that are not managed by Rook but
participate in consensus.

### Install the Arbiter Operator via Helm

```bash
helm install --create-namespace --namespace arbiter-operator \
  --values ./contrib/charts/external-arbiter-operator/local.yaml \
  arbiter-operator ./contrib/charts/external-arbiter-operator
```

### Configure the Remote Cluster

Create a namespace, user, role, rolebinding, kubeconfig, and secret for the
arbiter:

```bash
./hack/configure-k8s-user.sh
```

Apply the remote cluster access secret and resources:

```bash
kubectl apply -f ./contrib/k8s/examples/secret.yaml -n arbiter-operator
kubectl apply -f ./contrib/k8s/examples/remote-cluster.yaml -n arbiter-operator
kubectl apply -f ./contrib/k8s/examples/remote-arbiter.yaml -n arbiter-operator
```

Watch until the arbiter is ready:

```bash
kubectl get remotearbiter -n arbiter-operator -w
```

Verify the arbiter has joined the Ceph monitor quorum:

```bash
kubectl exec deployment/rook-ceph-tools -n rook-ceph -it -- ceph mon dump
```

---

## Step 5: Install OpenStack

CobaltCore deploys OpenStack services using Helm charts. The following
services are deployed as part of a standard installation:

**[TODO]:** Confirm the full list of OpenStack services and their Helm chart
sources for CobaltCore. The following is based on the architecture
documentation and may be incomplete.

### Add Helm Repositories

**[TODO]:** Confirm the Helm repository URLs for CobaltCore's OpenStack charts.

```bash
helm repo add <cobaltcore-openstack-repo> <repo-url>
helm repo update
```

### Install OpenStack Services

**[TODO]:** Confirm the exact Helm chart names, values files, and install
order for each OpenStack service. The following is a placeholder structure:

```bash
for service in keystone glance nova neutron horizon; do
  echo "Installing $service:"
  helm upgrade --install -n "$COBALTCORE_NAMESPACE" \
    "$service" <cobaltcore-openstack-repo>/"$service"
done
```

---

## Step 6: Deploy the Hypervisor Operator

The [Hypervisor Operator](https://cobaltcore-dev.github.io/docs/architecture/cluster.html#hypervisor-operator)
manages the lifecycle of hypervisor nodes, ensuring newly discovered nodes are
properly configured and integrated into the cluster.

**[TODO]:** Confirm the Helm chart name and repository for the Hypervisor
Operator.

```bash
helm upgrade --install -n "$COBALTCORE_NAMESPACE" \
  openstack-hypervisor-operator \
  <cobaltcore-repo>/openstack-hypervisor-operator
```

Verify the operator is running:

```bash
kubectl -n "$COBALTCORE_NAMESPACE" get pods -l app=openstack-hypervisor-operator
```

---

## Step 7: Deploy the KVM HA Service

The [KVM HA Service](https://cobaltcore-dev.github.io/docs/architecture/cluster.html#ha-service)
monitors the health and status of hypervisor nodes and their virtual machines,
ensuring that critical workloads remain operational in the event of failures.

**[TODO]:** Confirm the Helm chart name and repository for the KVM HA Service.

```bash
helm upgrade --install -n "$COBALTCORE_NAMESPACE" \
  kvm-ha-service \
  <cobaltcore-repo>/kvm-ha-service
```

## Step 8: Dashboards

### Background

CobaltCore provides two distinct dashboard options for monitoring and managing
the storage cluster, targeting different types of users.

The upstream **Ceph Dashboard** is well-integrated into the Ceph project and
works well for experienced users who are already familiar with Ceph and prefer
a GUI over the CLI. However, it was built primarily as a migration of CLI
functionality into a graphical interface, and has been found to be less
accessible to users who are new to Ceph or who prefer a more guided,
use-case-oriented interface.

To address this, CobaltCore provides a second, **simplified GUI** designed
specifically for inexperienced or visually oriented users. Rather than exposing
all possible configuration options, this interface is designed to guide users
through their goals step by step — asking what they want to achieve rather
than how. It is implemented using Vue.js and Naive UI, and is designed to run
outside the Ceph cluster. 

The two dashboards are complementary rather than competing. They serve
different needs and different audiences, and can be used alongside each other.

### Accessing the Ceph Dashboard

The Ceph Dashboard is deployed as part of the Rook-managed Ceph cluster. To
access it, retrieve the dashboard password and port-forward to the service:

```bash
# Get the dashboard password
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath="{['data']['password']}" | base64 --decode && echo

# Port-forward to access the dashboard
kubectl -n rook-ceph port-forward service/rook-ceph-mgr-dashboard 8443:8443
```

Access the dashboard at `https://localhost:8443` using the username `admin`
and the password retrieved above.

### Accessing the Simplified GUI

**[TODO]:** Confirm the deployment method, service name, and access URL for
the CobaltCore simplified GUI.

---

## Step 9: Verify the Installation

Check that all pods are running:

```bash
kubectl -n "$COBALTCORE_NAMESPACE" get pods
kubectl -n rook-ceph get pods
kubectl -n arbiter-operator get pods
```

Check the Ceph cluster health:

```bash
kubectl -n rook-ceph exec -it deployment/rook-ceph-tools -- ceph status
```

---

## Next Steps

After successful installation:

1. Review and adjust Ceph pool and storage class configuration
2. Review the other CobaltCore project components

---

## Additional Resources

- [CobaltCore Architecture](https://cobaltcore-dev.github.io/docs/architecture/)
- [CobaltCore GitHub Organisation](https://github.com/cobaltcore-dev)
- [IronCore Documentation](https://ironcore.dev/)
- [Rook Documentation](https://rook.io/docs/rook/latest-release/Getting-Started/intro/)
- [Gardener Documentation](https://gardener.cloud/docs/)
- [Cortex Repository](https://github.com/cobaltcore-dev/cortex)
- [Arbiter Repository](https://github.com/cobaltcore-dev/external-arbiter-operator)
