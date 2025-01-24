# Setup work agent dashbaord

## Installation

### Prerequisites
- Ensure the Klusterlet agent metrics are exposed to Prometheus. Refer to the guide [Expose Klusterlet agent metrics](../../guide/Metrics/KlusterletAgent.md) for detailed instructions.
- Grafana must be installed on the cluster where the Klusterlet agent is running, and a datasource should be properly configured for the local Prometheus. Follow the steps in [Install Grafana on an OpenShift cluster](../InstallGrafana.md) for setup guidance.

### Load dashboard from JSON file
1. Log in to the Grafana console and navigate to `Home > Dashboard`;
2. Click `New` and choose `Import` from the dropdown menu;
3. In the `Import dashboard` view, paste the contenet of [WorkAgent.json](WorkAgent.json) into the text area labeled `Import via dashboard JSON model`.
4. Click `Load` and then `Import` to load the dashboard.

## Work agent dashboard
The `Work Agent` dashboard is organized into three distinct sections, each designed to monitor specific Service Level Indicators (SLIs).

### Requests to Hub Cluster
#### Request Rate
This visualization displays the rate of requests sent by the Klusterlet agent to the hub cluster kube apiserver. A sustained high value may indicate that the Klusterlet agent is putting excessive pressure on the hub cluster Kube apiserver.

#### Request Duration - P75
This visualization shows the 75th percentile of the duration of requests sent by the Klusterlet agent to the hub cluster Kube apiserver. A consistently high value may indicate:
- The hub cluster Kube apiserver is experiencing latency.
- Client-side throttling, especially if the request rate is also high.

### Requests to Managed Cluster
#### Request Rate
The visualization shows the rate of the requests sent by the Klusterlet agent to the managed cluster Kube apiserver. If the value keeps large for a while, it may indicate the Klusterlet agent bring too much presure to the managed Kube apiserver.

This visualization displays the rate of requests sent by the Klusterlet agent to the managed cluster Kube apiserver. A sustained high value may indicate that the Klusterlet agent is placing excessive pressure on the managed cluster Kube apiserver.

#### Request Duration - P75
This visualization shows the 75th percentile of the duration of requests sent by the Klusterlet agent to the managed cluster Kube apiserver. A consistently high value indicates that the managed cluster Kube apiserver may be slow or overloaded.

### Controller Queue
#### Enqueue Rate
This visualization displays the enqueue rate of the work agent. A high value suggests that a large number of ManifestWorks are being created or modified.

#### Queue Depth
This visualization shows the number of ManifestWorks that remain unprocessed by the work agent. Under normal circumstances, this value should be close to zero or relatively low. A significantly high value suggests a backlog in the work agent processing queue.