# The `ocm-controller` is stuck in a CrashLoopBackOff state.

## Symptom
You're seeing errors like this in the logs of your `ocm-controller` pod in the `multicluster-engine` namespace:
```
failed to list *v1beta2.ManagedClusterSet: request to convert CR from an invalid group/version: cluster.open-cluster-management.io/v1beta1
```
or
```
failed to list *v1beta2.ManagedClusterSetBinding: request to convert CR from an invalid group/version: cluster.open-cluster-management.io/v1beta1
```
When you try to list `ManagedClusterSet` or `ManagedClusterSetBinding` using the command line tool, you might get this error:
```
Error from server: request to convert CR from an invalid group/version: cluster.open-cluster-management.io/v1beta1
```

## Meaning
The storage version migration (from v1beta1 to v1beta2) for `ManagedClusterSet` and `ManagedClusterSetBinding` may not have completed successfully, but the v1beta1 API has been removed.

## Impact
If this issue occurs, users and controllers will not be able to access the `ManagedClusterSet` and `ManagedClusterSetBinding` APIs. Some Multicluster Engine (MCE) components, like `ocm-controller`, will keep crashing and restarting.

## Mitigation
1. Pause MCE:
    ```
    oc annotate mce multiclusterengine pause=true --overwrite
    ```

2. Scale down the cluster-manager deployment to 0 replicas:
    ```
    oc -n multicluster-engine scale deploy cluster-manager --replicas=0
    ```

3. Apply CRDs that contain both v1beta1 and v1beta2 APIs:
    ```bash
    curl -s https://raw.githubusercontent.com/stolostron/ocm/refs/heads/backplane-2.4/manifests/cluster-manager/hub/0000_00_clusters.open-cluster-management.io_managedclustersets.crd.yaml | sed '6,18d' | oc apply -f -
    oc apply -f https://raw.githubusercontent.com/stolostron/ocm/refs/heads/backplane-2.4/manifests/cluster-manager/hub/0000_01_clusters.open-cluster-management.io_managedclustersetbindings.crd.yaml
    ```

4. Verify that you can list `managedclusterset` and `managedclustersetbinding` without errors:
    ```
    oc get managedclusterset
    oc get managedclustersetbinding -A
    ```

5. List the failed StorageVersionMigrations.
    ```
    oc get storageversionmigration -o jsonpath='{range .items[?(@.status.conditions[*].status=="False")]}{.metadata.name}{"\n"}{end}' | grep managedclusterset
    ```

6. Reset the status of each failed StorageVersionMigration to trigger another migration attempt.
    ```bash
    oc patch StorageVersionMigration <migration-name> --type='json' --subresource status -p='[{"op":"remove", "path":"/status/conditions"}]'
    ```

7. Wait for the migration to complete:
    ```
    oc wait storageversionmigration <migration-name> --for=condition=Succeeded --timeout=120s
    ```

8. Once all migrations are done, restore the CRDs and the cluster-manager deployment replicas by resuming MCE.
    ```
    oc annotate mce multiclusterengine pause- --overwrite
    ```