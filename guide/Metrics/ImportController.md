# Expose import-controller metrics
# Instructions
Run the following commands on the hub cluster:
1. Create a service to expose the metric endpoint of the import-controller.
```
oc apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: managedcluster-import-controller-v2
  name: import-controller
  namespace: multicluster-engine
spec:
  ports:
  - name: http
    port: 8383
    protocol: TCP
    targetPort: 8383
  selector:
    app: managedcluster-import-controller-v2
EOF
```

2. Create a ServiceMonitor to scrape the metrics of import-controller
```
oc apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: import-controller
  namespace: multicluster-engine
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 60s
    port: http
    scheme: http
    scrapeTimeout: 10s
  jobLabel: import-controller
  namespaceSelector:
    matchNames:
    - multicluster-engine
  selector:
    matchLabels:
      app: managedcluster-import-controller-v2
EOF
```

# Useful metrics
- controller_runtime_reconcile_total
- controller_runtime_reconcile_time_seconds_bucket
- rest_client_requests_total
- workqueue_adds_total
- workqueue_depth
