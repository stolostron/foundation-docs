# Expose Klusterlet agent metrics
# Instructions
Run the following commands on the cluster where the Klusterlet agent is running:
1. Add additional permissions for the `klusterlet-work-sa` ServiceAccount.
- ClusterRole
```
oc apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: klusterlet-tokenreviews
rules:
- apiGroups: ["authentication.k8s.io"]
  resources: ["tokenreviews"]
  verbs: ["create"]
EOF
```
- ClusterRoleBinding
```
oc apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: klusterlet-agent-tokenreviews
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: klusterlet-tokenreviews
subjects:
- kind: ServiceAccount
  name: klusterlet-work-sa
  namespace: open-cluster-management-agent
EOF
```

2. Add `cluster-monitoring` label to the `open-cluster-management-agent` namespace.
```
oc label --overwrite ns open-cluster-management-agent openshift.io/cluster-monitoring=true
```

3. Create a service to expose the metric endpoint of the Klusterlet agent
```
oc apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: klusterlet-agent
  name: klusterlet-agent
  namespace: open-cluster-management-agent
spec:
  ports:
  - name: https
    port: 8443
    protocol: TCP
    targetPort: 8443
  selector:
    app: klusterlet-agent
EOF
```

4. Create a ServiceMonitor to scrape the metrics of Klusterlet agent
```
oc apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: klusterlet-agent
  namespace: open-cluster-management-agent
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 60s
    port: https
    scheme: https
    scrapeTimeout: 10s
    tlsConfig:
      insecureSkipVerify: true
  jobLabel: klusterlet-agent
  namespaceSelector:
    matchNames:
    - open-cluster-management-agent
  selector:
    matchLabels:
      app: klusterlet-agent
EOF
```

# Useful metrics
- rest_client_requests_total
- rest_client_request_duration_seconds_bucket
- workqueue_depth

# Troubleshooting
## [Missing Permission for Prometheus Service Account](Toubleshooting-MissingPermissionForPrometheus.md)