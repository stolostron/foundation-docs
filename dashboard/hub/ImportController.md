# Setup Import Controller dashboard

## Installation

### Prerequisites
- Ensure the import-controller metrics are exposed to Prometheus. Refer to the guide [Expose import-controller metrics](../../guide/Metrics/ImportController.md) for detailed instructions.
- Grafana must be installed on the hub cluster, and a datasource should be properly configured for the local Prometheus. Follow the steps in [Install Grafana on an OpenShift cluster](../InstallGrafana.md) for setup guidance.

### Load dashboard from JSON file
1. Log in to the Grafana console and navigate to `Home > Dashboard`;
2. Click `New` and choose `Import` from the dropdown menu;
3. In the `Import dashboard` view, paste the contenet of [ImportController.json](ImportController.json) into the text area labeled `Import via dashboard JSON model`.
4. Click `Load` and then `Import` to load the dashboard.

## Import Controller dashboard
The `Import Controller` dashboard is organized into three distinct sections, each designed to monitor specific Service Level Indicators (SLIs).

### Managed Clusters
#### Number of managed clusters
This visualization displays the number of the managed clusters.

#### Derivative of managed cluster number
This visualization shows how quickly the number of managed clusters is increasing or decreasing during a given time window (per second).

### Requests
#### Request rate to hub cluster
This visualization displays the rate of requests sent by the import-controller to the hub cluster Kube apiserver. A sustained high value may indicate that the import-controller is placing excessive pressure on the hub cluster Kube apiserver.

### Controllers
#### Enqueue rate
This visualization displays the enqueue rate of the controllers. A high value suggests that a large number of ManagedCluster resources are being created or modified.

#### Queue depth
This visualization shows the number of ManagedCluster that remain unprocessed by the controllers. Under normal circumstances, this value should be close to zero or relatively low. A significantly high value suggests a backlog in the import-controller processing queue.

#### P75 - Reconciliation duration (second)
This visualization displays the 75th percentile of the reconciliation duration for controllers. Higher values suggest that the controller took an unusually long time to process a single managed cluster.

#### Error rate
This visualization shows the error rate of the controllers. A consistently high value may indicate internal errors occurring while processing the managed clusters.
