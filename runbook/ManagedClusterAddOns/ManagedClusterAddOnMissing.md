# The `ManagedClusterAddon` resource is not present in the cluster namespace

## Symptom
For addons such as `work-manager`, `clustr-proxy` and `managed-serviceaccount`, the `ManagedClusterAddon`
resource is missing in one or several clusters. You can check with command below:

```shell
oc get managedclusteraddon -n <cluster-name> -o <addon-name>
```
and it returns empty result.

## Meaning
The addon is not successfully installed in the cluster.

## Impact
Once the issue happens, the addon does not function properly. For instance, the URL of the managed cluster
or the `ClusterClaim` will not be populated to the `ManagedCluster` resource when the work-manager addon is not
working, the `ManagedClusterInfo` resource is not updated either.

## Diagnosis

Check if the `ManagedClusterAddon` resource existing in the cluster namespace.

```shell
oc get managedclusteraddon -n <cluster-name> -o <addon-name>
```
If the resource is missing. Check if global `ManagedClusterSet` exists:
```shell
oc get managedclusterset global
```
Check if namespace `open-cluster-management-global` exists:
```shell
oc get ns open-cluster-management-global
```
if the namespace exists, check if global `ManagedClusterBinding` and `Placement` exists:
```shell
oc get managedclustersetbinding -n open-cluster-management-global
```
and
```shell
oc get placement -n open-cluster-management-global
```
If the namespace, managedclusterbinding or placement does not exist, check global managedclusterset
```shell
oc get managedclusterset global -o yaml
```
and if the annotation `open-cluster-management.io/ns-create: "true"` is in the managedclusterset, need to
remove this annotation to make namespace/managedclustersetbinding/placement recreated.
