# Troubleshooting - Missing Permission for Prometheus
If the metrics are still unavailable after the required `Service` and `ServiceMonitor` resources are created, check the logs of the `prometheus` pod in the `openshift-monitoring` namespace using the following command:
```
oc -n openshift-monitoring logs prometheus-k8s-0
```
If you encounter the following errors,
```
ts=2025-02-11T03:57:48.994Z caller=klog.go:116 level=error component=k8s_client_runtime func=ErrorDepth msg="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:556: Failed to watch *v1.Pod: failed to list *v1.Pod: pods is forbidden: User \"system:serviceaccount:openshift-monitoring:prometheus-k8s\" cannot list resource \"pods\" in API group \"\" in the namespace \"open-cluster-management-agent\""
ts=2025-02-11T03:58:22.375Z caller=klog.go:108 level=warn component=k8s_client_runtime func=Warningf msg="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:554: failed to list *v1.Endpoints: endpoints is forbidden: User \"system:serviceaccount:openshift-monitoring:prometheus-k8s\" cannot list resource \"endpoints\" in API group \"\" in the namespace \"open-cluster-management-agent\""
ts=2025-02-11T03:58:28.619Z caller=klog.go:116 level=error component=k8s_client_runtime func=ErrorDepth msg="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:555: Failed to watch *v1.Service: failed to list *v1.Service: services is forbidden: User \"system:serviceaccount:openshift-monitoring:prometheus-k8s\" cannot list resource \"services\" in API group \"\" in the namespace \"open-cluster-management-agent\""
```
grant the missing permissions to the `prometheus-k8s` service account.
```
oc apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s-ext
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - services
  - pods
  - endpoints
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-k8s-ext
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-k8s-ext
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: openshift-monitoring
EOF
```