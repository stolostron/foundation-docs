# Auto-import of `ManagedCluster` not triggered

## Symptom
The `ManagedCluster` on the hub cluster has a condition of type `ManagedClusterConditionAvailable` with a `status` of `Unknown`, while the admin-kubeconfig referenced by the `ClusterDeployment` resource is present.

## Meaning
The auto-import of this managed cluster was not completed successfully.

## Diagnosis

Check if the names of `ClusterDeployment` and `ManagedCluster` match. The names of these two resources must be the same.
```
oc get managedcluster <cluster-name>
oc -n <cluster-name> get clusterdeployment
```
Verify the `spec.installed` field of the `ClusterDeployment`. Ensure that `spec.installed` is set to `true`. The auto-import process will not start until the cluster is successfully provisioned (`spec.installed` becomes `true`).
```
oc -n <cluster-name> get clusterdeployment <cluster-name> -o jsonpath='{.spec.installed}'
```
## Mitigation

### Import the cluster manually
If the auto-import is not working, you can manually import the cluster by following the instructions from [Import cluster manually with CLI tool](../../guide/ManagedCluster/ManagedClusterManualImport.md).

## Required artifacts for further troubleshooting
If none of the above runbooks helps, please collect the ACM must-gather data of both hub cluster and the managed cluster for troubleshooting. If the must-gather data is not available for some reason, please collect the following information instead.
- On the hub cluster
  - YAML of the `ManagedCluster`
    ```
    oc get managedcluster <cluster-name> -o yaml > cluster.yaml
    ```
  - YAML of the `ClusterDeployment`
    ```
    oc -n <cluster-name> get clusterdeployment -o yaml > clusterdeployment.yaml
    ```
  - Log of the import controller
    ```
    oc -n multicluster-engine logs -l app=managedcluster-import-controller-v2 > import-controller.log
    ```
