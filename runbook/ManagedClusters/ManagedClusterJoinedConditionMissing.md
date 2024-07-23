# The `ManagedClusterJoined` condition of `ManagedCluster` is not present

## Symptom
The ManagedCluster on the hub cluster has no condition of type `ManagedClusterJoined`. You are able to check the status of this condition with command below,
```
oc get managedcluster <cluster-name> -o jsonpath='{.status.conditions[?(@.type=="ManagedClusterJoined")]}'
```

## Meaning
The registration of this managed cluster is not finished sucessfully.

## Impact
Once this issue happens, the status of the condition `ManagedClusterConditionAvailable` for this managed cluster will become `Unknown` eventually. You are able to check the status of this condition with command below,
```
oc get managedcluster <cluster-name> -o jsonpath='{.status.conditions[?(@.type=="ManagedClusterConditionAvailable")].status}'
```

## Diagnosis

Check the `status.conditions` of the `ManagedCluster` on the hub cluster.
```
oc get managedcluster <cluster-name> -o yaml
```
Refers to the runbooks below for the diagnosis instructions if any condition matches.
- [The `ManagedClusterImportSucceeded` condition of `ManagedCluster` is `False`](./ManagedClusterImportSucceededConditionFalse.md)

On the managed cluster, check the existance of the resources below.
- The `Klusterlet` resource, with command `oc get klusterlet klusterlet`;
- The Klusterlet operator, with command `oc -n open-cluster-management-agent get pod -l app=klusterlet`

If any of those resources is missing, you are able to recover them with the instructions from [Mitigation -> Reinstall the klusterlet](#reinstall-the-klusterlet)

Check the status of the `Klusterlet` resource on the managed cluster and see if there is any error in the conditions.
```
oc get klusterlet klusterlet -o yaml
```

Typical errros:
- [bootstrap hub kubeconfig is degraded](...)
- ... ...

Check the log of the klusterlet-agent on the managed cluster and see if there is any error.
```
oc -n open-cluster-management-agent logs -l app=klusterlet-agent
```

Typical errros:
- [connection timeout](...)
- [X509](...)
- ... ...

## Mitigation

### Reinstall the klusterlet
To reinstall the klusterlet on the managed cluster, follow the instructions from [Import cluster manually with CLI tool](../../guide/ManagedCluster/ManagedClusterManualImport.md).

## Required artifacts for further troubleshooting
If none of the above runbooks helps, please collect the ACM must-gather data of both hub cluster and the managed cluster for troubleshooting. If the must-gather data is not available for some reason, please collect the following information instead.
- On the hub cluster
  - YAML of the `ManagedCluster`
    ```
    oc get managedcluster <cluster-name> -o yaml > cluster.yaml
    ```
  - Log of the import controller
    ```
    oc -n multicluster-engine logs -l app=managedcluster-import-controller-v2 > import-controller.log
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
  - Log of klusterlet pod;
    ```
    oc -n open-cluster-management-agent logs -l app=klusterlet > klusterlet.log
    ```
  - Log of klusterlet-agent pod;
    ```
    oc -n open-cluster-management-agent logs -l app=klusterlet-agent > klusterlet-agent.log
    ```
