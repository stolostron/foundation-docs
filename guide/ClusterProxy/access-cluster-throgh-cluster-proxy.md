# Accessing Managed Clusters through the Cluster-Proxy Add-on

The **cluster-proxy add-on** enables secure access from the hub to services running on managed clusters.
To connect, you need:

* **Endpoint** – Proxy server address
* **CA bundle** – To verify certificates
* **Token** – Required if the service requires authentication

---

## Accessing the kube-apiserver of a Managed Cluster

### Preparation

1. **Select a managed cluster**

```bash
export MANAGED_CLUSTER_NAME=cluster1
```

2. **Get the proxy server endpoint**

```bash
export PROXY_SERVER_ENDPOINT=$(kubectl get route -n multicluster-engine cluster-proxy-addon-user -o=jsonpath='{.spec.host}')/$MANAGED_CLUSTER_NAME
```

3. **Retrieve the CA bundle**

```bash
kubectl get configmap -n openshift-service-ca kube-root-ca.crt \
  -o=go-template='{{index .data "ca.crt"}}' > ca.crt
```

4. **Create a ManagedServiceAccount**

```bash
kubectl apply -f msa.yaml
```

**msa.yaml**

```yaml
apiVersion: authentication.open-cluster-management.io/v1beta1
kind: ManagedServiceAccount
metadata:
  name: my-sa1
  namespace: cluster1
spec:
  rotation: {}
```

5. **Fetch the token**

```bash
export MANAGED_SA_NAME=my-sa1
export MANAGED_SA_TOKEN=$(kubectl -n $MANAGED_CLUSTER_NAME get secret $MANAGED_SA_NAME -o jsonpath={.data.token} | base64 -d)
```

6. **Grant permissions** (required before using the token)

```bash
kubectl apply -f permission.yaml
```

**permission.yaml**

```yaml
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: my-sa1-permissions
  namespace: cluster1
spec:
  roles:
  - namespace: open-cluster-management-agent
    rules:
    - apiGroups: [""]
      resources: ["pods", "pods/log", "events", "services"]
      verbs: ["get","list"]
  roleBindings:
  - namespace: open-cluster-management-agent
    roleRef:
      kind: Role
    subject:
      apiGroup: authentication.open-cluster-management.io
      kind: ManagedServiceAccount
      name: my-sa1
```

Now you have:

* `PROXY_SERVER_ENDPOINT`
* `ca.crt`
* `MANAGED_SA_TOKEN` (with permissions)

---

## Access Kubernetes Resources

### Using `kubectl`

1. **Build a kubeconfig**

```bash
export KUBECONFIG=./kubeconfig.$MANAGED_CLUSTER_NAME
kubectl config set-cluster my-cluster \
  --server=https://$PROXY_SERVER_ENDPOINT \
  --certificate-authority=./ca.crt --embed-certs=true
kubectl config set-credentials my-user --token=$MANAGED_SA_TOKEN
kubectl config set-context my-context --cluster=my-cluster --user=my-user
kubectl config use-context my-context
```

2. **Run commands**

```bash
kubectl -n open-cluster-management-agent get pods
```

---

### Using `curl`

```bash
curl --cacert ./ca.crt -H "Authorization: Bearer $MANAGED_SA_TOKEN" \
  https://$PROXY_SERVER_ENDPOINT/api/v1/namespaces/open-cluster-management-agent/pods
```

---

## Accessing Other Services on a Managed Cluster

The cluster-proxy add-on can also route to **any HTTP/HTTPS service** running in a managed cluster, not just the kube-apiserver.

Example: accessing the `clusterlifecycle-state-metrics-v2` service in the `multicluster-engine` namespace on the `local-cluster`.

> This service does not require authentication, so no token is needed.

---

### Steps

1. **Select the managed cluster**

```bash
export MANAGED_CLUSTER_NAME=local-cluster
```

2. **Get the proxy server endpoint**

```bash
export PROXY_SERVER_ENDPOINT=$(kubectl get route -n multicluster-engine cluster-proxy-addon-user -o=jsonpath='{.spec.host}')/$MANAGED_CLUSTER_NAME
```

3. **Retrieve the CA bundle**

```bash
kubectl get configmap -n openshift-service-ca kube-root-ca.crt \
  -o=go-template='{{index .data "ca.crt"}}' > ca.crt
```

4. **Register the service with a ManagedProxyServiceResolver**

```bash
kubectl apply -f resolver.yaml
```

**resolver.yaml**

```yaml
apiVersion: proxy.open-cluster-management.io/v1alpha1
kind: ManagedProxyServiceResolver
metadata:
  name: metrics-service
spec:
  managedClusterSelector:
    managedClusterSet:
      name: global
    type: ManagedClusterSet
  serviceSelector:
    serviceRef:
      name: clusterlifecycle-state-metrics-v2
      namespace: multicluster-engine
    type: ServiceRef
```

5. **Access the service**

```bash
curl --cacert ./ca.crt \
  https://$PROXY_SERVER_ENDPOINT/api/v1/namespaces/multicluster-engine/services/https:clusterlifecycle-state-metrics-v2:8443/proxy-service/metrics
```

---

✅ With these steps, you can securely access both the **kube-apiserver** and any other HTTP/HTTPS services in managed clusters through the cluster-proxy add-on.