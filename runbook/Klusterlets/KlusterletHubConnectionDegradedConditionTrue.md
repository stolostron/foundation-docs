# The `HubConnectionDegraded` condition of `klusterlet` is `True`

## Symptom
The `HubConnectionDegraded` condition of `klusterlet` on the managed cluster is `True`. You are able to check the condition with command below,
```
oc get klusterlet klusterlet -o jsonpath='{.status.conditions[?(@.type=="HubConnectionDegraded")]}'
```

## Meaning
The klusterlet running on the managed cluster has trouble when it tries to access the hub cluster cluster.

## Impact
The klusterlet can not access the hub cluster.
Once this issue happens, the status of the condition `ManagedClusterConditionAvailable` for this managed cluster will become `Unknown` eventually. You are able to check the status of this condition with command line below,
```
oc get managedcluster <cluster-name> -o jsonpath='{.status.conditions[?(@.type=="ManagedClusterConditionAvailable")].status}'
```

## Diagnosis
Check the reason of the `HubConnectionDegraded` condition.
```
oc get klusterlet klusterlet -o yaml
```
Typical reasons:
- `BootstrapSecretError`
- `HubKubeConfigSecretMissing`

### BootstrapSecretError

#### tls: failed to verify certificate: x509: certificate signed by unknown authority
```
  conditions:
  - lastTransitionTime: "2023-12-18T14:23:00Z"
    message: |-
      Failed to create &SelfSubjectAccessReview{ObjectMeta:{      0 0001-01-01 00:00:00 +0000 UTC <nil> <nil> map[] map[] [] [] []},Spec:SelfSubjectAccessReviewSpec{ResourceAttributes:&ResourceAttributes{Namespace:,Verb:create,Group:cluster.open-cluster-management.io,Version:,Resource:managedclusters,Subresource:,Name:,},NonResourceAttributes:nil,},Status:SubjectAccessReviewStatus{Allowed:false,Reason:,EvaluationError:,Denied:false,},} with bootstrap secret "open-cluster-management-agent" "bootstrap-hub-kubeconfig": Post "https://api.hs-sc-eodbuc6c0.jlf6.i1.devshift.org:6443/apis/authorization.k8s.io/v1/selfsubjectaccessreviews": tls: failed to verify certificate: x509: certificate signed by unknown authority
      Failed to get hub kubeconfig secret "open-cluster-management-agent" "hub-kubeconfig-secret": secrets "hub-kubeconfig-secret" not found
    observedGeneration: 1
    reason: BootstrapSecretError,HubKubeConfigSecretMissing
    status: "True"
    type: HubConnectionDegraded
```

#### Unauthorized
```
  conditions:
  - lastTransitionTime: "2023-06-26T17:04:32Z"
      message: |-
        Failed to create &SelfSubjectAccessReview{ObjectMeta:{      0 0001-01-01 00:00:00 +0000 UTC <nil> <nil> map[] map[] [] [] []},Spec:SelfSubjectAccessReviewSpec{ResourceAttributes:&ResourceAttributes{Namespace:,Verb:create,Group:cluster.open-cluster-management.io,Version:,Resource:managedclusters,Subresource:,Name:,},NonResourceAttributes:nil,},Status:SubjectAccessReviewStatus{Allowed:false,Reason:,EvaluationError:,Denied:false,},} with bootstrap secret "open-cluster-management-agent" "bootstrap-hub-kubeconfig": Unauthorized
        Failed to get hub kubeconfig secret "open-cluster-management-agent" "hub-kubeconfig-secret": secrets "hub-kubeconfig-secret" not found
      observedGeneration: 4
      reason: BootstrapSecretError,HubKubeConfigSecretMissing
      status: "True"
      type: HubConnectionDegraded
```

### HubKubeConfigSecretMissing
```
conditions:
- lastTransitionTime: "2023-04-05T14:30:48Z"
  message: |-
    Bootstrap secret open-cluster-management-agent/bootstrap-hub-kubeconfig to apiserver https://api.pdcocp4ctl01.prod.local:6443 is configured correctly
    Failed to get hub kubeconfig secret "open-cluster-management-agent" "hub-kubeconfig-secret": secrets "hub-kubeconfig-secret" not found
  observedGeneration: 5
  reason: BootstrapSecretFunctional,HubKubeConfigSecretMissing
  status: "True"
  type: HubConnectionDegraded
```

## Mitigation

### Reinstall the klusterlet
To reinstall the klusterlet on the managed cluster, follow the instructions from [Import cluster manually with CLI tool](../../guide/ManagedCluster/ManagedClusterManualImport.md).

## Required artifacts
If none of the above runbooks helps, please collect the ACM must-gather data of both hub cluster and the managed cluster for troubleshooting. If the must-gather data is not available for some reason, please collect the following information instead.
- On the hub cluster
  - YAML of the `ManagedCluster`
    ```
    oc get managedcluster <cluster-name> -o yaml > cluster.yaml
    ```
  - import.yaml of the managed cluster
    ```
    oc -n <cluster-name> get secret <cluster-name>-import -o jsonpath='{.data.import\.yaml}' | base64 -d > import.yaml
    ```
- On the managed cluster
  - YAML of `Klusterlet`;
    ```
    oc get klusterlet klusterlet -o yaml > klusterlet.yaml
    ```
  - Pod list in the `open-cluster-management-agent` namespace;
    ```
    oc -n open-cluster-management-agent get pods > agent-pods.txt
    ```
  - Secret list in the `open-cluster-management-agent` namespace;
    ```
    oc -n open-cluster-management-agent get secrets > agent-secrets.txt
    ```
  - Bootstrap hub kubeconfig;
    ```
    oc -n open-cluster-management-agent get secret bootstrap-hub-kubeconfig -o jsonpath='{.data.kubeconfig}' | base64 -d > bootstrap-hub-kubeconfig
    ```
  - Hub kubeconfig;
    ```
    oc -n open-cluster-management-agent get secret hub-kubeconfig-secret -o jsonpath='{.data.kubeconfig}' | base64 -d > hub-kubeconfig
    ```
  - Log of klusterlet pod;
    ```
    oc -n open-cluster-management-agent logs -l app=klusterlet > klusterlet.log
    ```
  - Log of klusterlet-agent pod;
    ```
    oc -n open-cluster-management-agent logs -l app=klusterlet-agent > klusterlet-agent.log
    ```
