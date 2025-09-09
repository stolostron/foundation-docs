# Kueue Addon Installation Guide

## Overview

The [Kueue Addon](https://github.com/open-cluster-management-io/addon-contrib/tree/main/kueue-addon) for Red Hat Advanced Cluster Management (ACM) allows a user to enable RHBoK on their managed clusters, simplifies MultiKueue setup and enhances multicluster batch job scheduling capabilities across OpenShift clusters. 

Key features include:
- **Simplified MultiKueue Setup**: Automates kubeconfig generation and streamlines MultiKueue resource configuration
- **Centralized Resource Management**: Manage spoke cluster resources from a single ACM hub cluster
- **Enhanced Multicluster Scheduling**: Integrates ACM placement with MultiKueue for intelligent workload distribution
- **Operator-based installation**: Allows a user to enable RHBoK on their managed clusters

## Prerequisites

Before installing the Kueue Addon, ensure the following requirements are met:

### ACM Hub Cluster Requirements
- Red Hat Advanced Cluster Management (ACM) installed and configured.
- The following ACM addons must be installed and available:
  - **[Cluster Permission Addon](https://github.com/open-cluster-management-io/cluster-permission)**: Enables proper RBAC for cross-cluster operations.
  - **[Managed Service Account Addon](https://github.com/open-cluster-management-io/managed-serviceaccount)**: Handles service account management across clusters.
  - **[Cluster Proxy Addon](https://github.com/open-cluster-management-io/cluster-proxy)**: Provides secure communication channels (optional but recommended).

### Kueue Requirements
- **Kueue operator already installed on the hub cluster** with MultiKueue functionality enabled.
- Spoke clusters accessible and registered with ACM.

## Installation

The Kueue Addon uses an operator-based installation approach, leveraging OpenShift's Operator Lifecycle Manager (OLM) to deploy RHBoK on spoke clusters.

### Step 1: Add the OCM Helm Repository

```bash
$ helm repo add ocm https://open-cluster-management.io/helm-charts/
$ helm repo update
$ helm search repo ocm/kueue-addon
NAME            CHART VERSION   APP VERSION     DESCRIPTION
ocm/kueue-addon <chart-version> <app-version>           A Helm chart for Open Cluster Management Kueue ...
```

### Step 2: Create Operator Configuration

Create a `values.operator.yaml` file with the following OpenShift-specific configuration:

```yaml
# This is the namespace where the Kueue controller is installed; the MultiKueue secret will be copied to this namespace
kueue:
  namespace: "openshift-kueue-operator"

# Install Kueue via Operator
installKueueViaOperator: true

# Operator Lifecycle Manager RBAC configuration
operatorLifecycleManager:
  clusterRoleBindingName: kueue-operator-lifecycle-manager-rolebinding
  clusterRoleName: system:controller:operator-lifecycle-manager

# Kueue Operator configuration
kueueOperator:
  name: kueue-operator
  namespace: openshift-kueue-operator
  operatorGroupName: openshift-kueue-operator
  channel: stable-v1.0
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: kueue-operator.v1.0.1

# Cert Manager Operator configuration
certManagerOperator:
  name: openshift-cert-manager-operator
  namespace: cert-manager-operator
  operatorGroupName: cert-manager-operator
  channel: stable-v1
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: cert-manager-operator.v1.17.0

# Kueue CR customization
kueueCR:
  spec:
    config:
      integrations:
        frameworks:
        - BatchJob
    managementState: Managed

# Cluster proxy configuration
clusterProxy:
  url: "https://<cluster-proxy-url>"
  impersonation:
    enabled: true

networkPolicy:
  name: kueue-allow-egress-cluster-proxy-dns
  namespace: "openshift-kueue-operator"
  spec:
    podSelector:
      matchLabels:
        app.openshift.io/name: kueue
    policyTypes:
    - Egress
    egress:
    - ports:
      - port: 80
        protocol: TCP
    - ports:
      - port: 443
        protocol: TCP
    - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: multicluster-engine
```

### Step 3: Install the Kueue Addon

Install the addon using Helm with the operator-specific configuration:

```bash
helm install \
  -n open-cluster-management-addon --create-namespace \
  kueue-addon ocm/kueue-addon \
  -f values.operator.yaml
```

## Verification Steps

After installation, verify that the Kueue Addon is properly deployed and functional:

### Validate Hub Cluster Installation

```bash
$ oc get cma multicluster-kueue-manager
NAME                         DISPLAY NAME                 CRD NAME
multicluster-kueue-manager   Multicluster Kueue Manager

$ oc get mca -A | grep kueue
NAMESPACE          NAME                          AVAILABLE   DEGRADED   PROGRESSING
<your cluster>     multicluster-kueue-manager    True                   False

$ oc get deploy -n open-cluster-management-addon kueue-addon-controller
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
kueue-addon-controller   1/1     1            1           4d19h

$ oc get multikueuecluster -A
NAME            CONNECTED   AGE
cluster1        True        4d19h
local-cluster   False       4d19h
```

**Note**: ACM automatically generates MultiKueueCluster resources for all managed clusters. However, notice that the `local-cluster` shows a CONNECTED status of `false`. This is expected behavior because MultiKueue currently doesn't support submitting jobs to the management cluster. For technical details, see the [MultiKueue design documentation](https://github.com/kubernetes-sigs/kueue/tree/main/keps/693-multikueue).

### Validate Spoke Cluster Installation

```bash
$ oc get subscriptions.operators.coreos.com -A
NAMESPACE                  NAME                              PACKAGE                           SOURCE             CHANNEL
cert-manager-operator      openshift-cert-manager-operator   openshift-cert-manager-operator   redhat-operators   stable-v1
openshift-kueue-operator   kueue-operator                    kueue-operator                    redhat-operators   stable-v1.0

$ oc get csv
NAME                            DISPLAY                                       VERSION   REPLACES                        PHASE
cert-manager-operator.v1.17.0   cert-manager Operator for Red Hat OpenShift   1.17.0    cert-manager-operator.v1.16.1   Succeeded
kueue-operator.v1.0.1           Red Hat build of Kueue                        1.0.1                                     Succeeded

$ oc get deployment kueue-controller-manager -n openshift-kueue-operator
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
kueue-controller-manager   2/2     2            2           6d18h

$ oc get clusterqueue
NAME            COHORT   PENDING WORKLOADS
cluster-queue            0

$ oc get localqueue
NAME         CLUSTERQUEUE    PENDING WORKLOADS   ADMITTED WORKLOADS
user-queue   cluster-queue   0                   0

$ oc get resourceflavor
NAME             AGE
default-flavor   4h28m
```

## Uninstall

To remove the Kueue Addon from the hub cluster:

```bash
helm uninstall -n open-cluster-management-addon kueue-addon
```

After uninstalling, the `Subscriptions` `ClusterServiceVersion` and `OperatorGroup` will be automatically removed from managed clusters.

However, the following Kueue resources will remain in the managed clusters.

Check remaining Kueue CR:

```bash
$ oc get kueue
NAME      AGE
cluster   42h
```

Check remaining Kueue CRDs:

```bash
$ oc get crd | grep kueue
admissionchecks.kueue.x-k8s.io                                            2025-09-16T13:47:07Z
clusterqueues.kueue.x-k8s.io                                              2025-09-16T13:47:07Z
kueues.kueue.openshift.io                                                 2025-09-16T13:47:00Z
localqueues.kueue.x-k8s.io                                                2025-09-16T13:47:07Z
multikueueclusters.kueue.x-k8s.io                                         2025-09-16T13:47:08Z
multikueueconfigs.kueue.x-k8s.io                                          2025-09-16T13:47:08Z
provisioningrequestconfigs.kueue.x-k8s.io                                 2025-09-16T13:47:08Z
resourceflavors.kueue.x-k8s.io                                            2025-09-16T13:47:08Z
workloadpriorityclasses.kueue.x-k8s.io                                    2025-09-16T13:47:09Z
workloads.kueue.x-k8s.io                                                  2025-09-16T13:47:09Z
```

**Note**: The Kueue CR and CRDs are left in the cluster after addon removal. If you need to completely remove Kueue from the managed clusters, you must manually delete these resources.

## Usage Scenarios

For detailed usage examples and advanced scenarios, refer to the [Upstream Kueue Integration Solution Usage Scenarios](https://github.com/open-cluster-management-io/ocm/tree/main/solutions/kueue-admission-check).