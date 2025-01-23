# Install Grafana on an OpenShift cluster

## 1. Create a namespace
```
oc create ns my-grafana
```

# 2. Install the Grafana operator
```
cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: grafana
  namespace: my-grafana
spec:
  targetNamespaces:
  - my-grafana
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: grafana-operator
  namespace: my-grafana
spec:
  channel: v5
  installPlanApproval: Automatic
  name: grafana-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
EOF
```

# 3. Setup RBAC
```
cat << EOF | oc apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana-sa
  namespace: my-grafana  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: edit-0
  namespace: my-grafana
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- kind: ServiceAccount
  name: grafana-sa
  namespace: my-grafana
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-monitoring-view-0
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-monitoring-view
subjects:
- kind: ServiceAccount
  name: grafana-sa
  namespace: my-grafana
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openshift-cluster-monitoring-view
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-cluster-monitoring-view
subjects:
- kind: ServiceAccount
  name: grafana-sa
  namespace: my-grafana
EOF
```

# 4. Create the Grafana credentials
```
cat << EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: grafana-sa
  name: grafana-credentials
  namespace: my-grafana  
type: kubernetes.io/service-account-token
---
apiVersion: batch/v1
kind: Job
metadata:
  name: patch-grafanadatasource
  namespace: my-grafana  
spec:
  template:
    spec:
      containers:
      - command:
        - /bin/bash
        - -c
        - |
          set -e
          NEW_TOKEN=`oc create token grafana-sa --duration=876000h -n my-grafana`
          oc set data secret/grafana-credentials token=`echo ${NEW_TOKEN}`
          oc delete pod -l app=grafana
        image: registry.redhat.io/openshift4/ose-cli:latest
        name: patch-grafanadatasource
      restartPolicy: Never
      serviceAccountName: grafana-sa
      terminationGracePeriodSeconds: 60
EOF
```

# 5. Create a Grafana instance
```
cat << EOF | oc apply -f -
apiVersion: grafana.integreatly.org/v1beta1
kind: Grafana
metadata:
  labels:
    dashboards: my-grafana
  name: grafana
  namespace: my-grafana
spec:
  config:
    auth:
      disable_login_form: "false"
    log:
      mode: console
    security:
      admin_password: redhat
      admin_user: admin
  route:
    spec:
      port:
        targetPort: grafana
      tls:
        termination: edge
      to:
        kind: Service
        name: grafana-service
        weight: 100
      wildcardPolicy: None
EOF
```

# 6. Setup the Grafana datasource for Prometheus
```
cat << EOF | oc apply -f -
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDatasource
metadata:
  name: prometheus-grafanadatasource
  namespace: my-grafana
spec:
  datasource:
    access: proxy
    editable: true
    isDefault: true
    jsonData:
      httpHeaderName1: Authorization
      timeInterval: 5s
      tlsSkipVerify: true
    name: Prometheus
    secureJsonData:
      httpHeaderValue1: Bearer \${token}
    type: prometheus
    url: https://thanos-querier.openshift-monitoring.svc.cluster.local:9091
  instanceSelector:
    matchLabels:
      dashboards: my-grafana
  valuesFrom:
  - targetPath: secureJsonData.httpHeaderValue1
    valueFrom:
      secretKeyRef:
        key: token
        name: grafana-credentials
EOF
```

# 7. Access the Grafana console with the Route URL
```
oc -n my-grafana get route
```