# The `ManagedClusterConditionAvailable` condition of `ManagedCluster` is `Unknown`

## Symptom
The `ManagedCluster` on the hub cluster has a condition of type `ManagedClusterConditionAvailable` with a `status` of `Unknown`.

You are able to check the status of this condition with command below,
```
oc get managedcluster <cluster-name> -o jsonpath='{.status.conditions[?(@.type=="ManagedClusterConditionAvailable")].status}'
```

## Meaning
When the klusterlet agent starts running on the managed cluster, it updates a `Lease` resource in the cluster namespace on the hub cluster every N seconds, where N is configured in the `spec.leaseDurationSeconds` of the `ManagedCluster`. If the `Lease` resource is not updated in the past 5 * N seconds, the status of the `ManagedClusterConditionAvailable` condition for this managed cluster will be set to `Unknown`. Once the klusterlet agent connects back to the hub cluster and continues to update the `Lease` resource, the ManagedCluster will become available automatically.

## Impact
Once this issue happens, 
- Usally both the klusterlet agent and add-on agents cannot connect to the hub cluster. Changes on ManifestWorks and other add-on specific resources on the hub cluster can not be pulled to the managed cluster;
- The status of the `Available` condition of all `ManagedClusterAddOns` for this managed cluster will be set to `Unknown` as well;
- The status of the `ManifestWorks` on the hub cluster is stale. You can not trust it anymore;

## Diagnosis
Check the `status.conditions` of the `ManagedCluster` on the hub cluster.
```
oc get managedcluster <cluster-name> -o yaml
```
Refers to the runbooks below for the diagnosis instructions if any condition matches.
- [The `ManagedClusterImportSucceeded` condition of `ManagedCluster` is `False`](./ManagedClusterImportSucceededConditionFalse.md)
- [The `ManagedClusterJoined` condition of `ManagedCluster` is not present](./ManagedClusterJoinedConditionMissing.md).
- [The `ManagedClusterConditionClockSynced` condition of `ManagedCluster` is `False`](./ManagedClusterConditionClockSyncedConditionFalse.md).

And then check the `status.conditions` of the `Klusterlet` on the managed cluster.
```
oc get klusterlet klusterlet -o yaml
```
If the command fails with timeout error, ensure the cluster is running and reachable; otherwise refers to the runbooks below for the diagnosis instructions.
  - [the `Klusterlet` resource does not exist](../Klusterlets/KlusterletNotInstalledOrIncomplete.md)
  - [The `HubConnectionDegraded` condition of `klusterlet` is `True`](../Klusterlets/KlusterletHubConnectionDegradedConditionTrue.md)
  - [The `Available` condition of `klusterlet` is `False`](../Klusterlets/KlusterletAvailableConditionFalse.md)

## Required artifacts for troubleshooting
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
