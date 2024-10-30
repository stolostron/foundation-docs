# The `ManagedClusterAddon` resource is not present in the cluster namespace

## Symptom
For addons such as `work-manager`, `clustr-proxy` and `managed-serviceaccount`, the `ManagedClusterAddon`
resource is missing in one or several clusters. You can check with command below:

```shell
oc get managedclusteraddon -n <cluster-name> -o <addon-name>
```
and it returns empty result.

## Meaning
The addon is not successfully installed in clusters.

## Impact
Once the issue happens, the addon does not function properly. For instance, the URL of the managed cluster
or the `ClusterClaim` will not be populated to the `ManagedCluster` resource when the work-manager addon is not
working, the `ManagedClusterInfo` resource is not updated either.

## Diagnosis

At first, check if the `ClusterManagementAddon` exists.
Check the `ClusterManagementAddon` on the hub cluster.
```shell
oc get clustermanagementaddon <addon-name> -o yaml
```

If the yaml exists, check the `spec.installStrategy` field. By default, the type will be
`Placements` with the following field:
```yaml
placements:
  - name: global
    namespace: open-cluster-management-global
```

Next check if the `ManagedClusterAddon` resource existing in the cluster namespace.
```shell
oc get managedclusteraddon <addon-name> -n <cluster-name>
```

If the resource is missing. Check if global `ManagedClusterSet` exists:
```shell
oc get managedclusterset global -o yaml
```
And check if the annotion `open-cluster-management.io/ns-create: "true"` is set already on the
`ManagedClusterSet` resource.

Then check if namespace `open-cluster-management-global` exists:
```shell
oc get ns open-cluster-management-global -o yaml
```

If the namespace exists, check if global `ManagedClusterBinding` and `Placement` exists:
```shell
oc get managedclustersetbinding global -n open-cluster-management-global -o yaml
```
and
```shell
oc get placement global -n open-cluster-management-global -o yaml
```

If the namespace, managedclusterbinding or placement does not exist, and the annotation
`open-cluster-management.io/ns-create: "true"` is set in the managedclusterset,
remove this annotation to make namespace/managedclustersetbinding/placement recreated.
