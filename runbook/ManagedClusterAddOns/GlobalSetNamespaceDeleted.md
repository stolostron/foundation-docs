# The `open-cluster-management-global-set` namespace is deleted causing addon uninstallation

## Symptom

All addons using the global placement install strategy (such as `managed-serviceaccount`, `work-manager`, `cluster-proxy`) are unexpectedly uninstalled from all managed clusters after the `open-cluster-management-global-set` namespace gets deleted.

You can verify this by checking that ManagedClusterAddOn resources are missing:

```shell
oc get managedclusteraddon -A | grep <addon-name>
```

## Meaning

When the `open-cluster-management-global-set` namespace is deleted, it breaks the global placement mechanism that these addons rely on for installation across all clusters.

## Impact

- All addons configured with global placement install strategy become non-functional
- Addon functionality is lost across the entire cluster fleet
- Services like managed service accounts, work management, and cluster proxy stop working
- ManagedClusterAddOn resources are removed from all cluster namespaces

## Diagnosis

1. Check if the `open-cluster-management-global-set` namespace exists:
```shell
oc get ns open-cluster-management-global-set
```

2. Verify the ClusterManagementAddOn configuration for affected addons:
```shell
oc get clustermanagementaddon managed-serviceaccount -o yaml
```

Look for the `installStrategy` field with placements pointing to the deleted namespace:
```yaml
spec:
  installStrategy:
    placements:
    - name: global
      namespace: open-cluster-management-global-set
      rolloutStrategy:
        type: All
    type: Placements
```

3. Check the global ManagedClusterSet status:
```shell
oc get managedclusterset global -o yaml
```

4. Verify if the annotation for namespace creation is present:
```shell
oc get managedclusterset global -o jsonpath='{.metadata.annotations.open-cluster-management\.io/ns-create}'
```

## Solution

To recover from this situation, trigger the recreation of the `open-cluster-management-global-set` namespace by removing the namespace creation annotation from the global ManagedClusterSet:

```shell
oc annotate managedclusterset global open-cluster-management.io/ns-create-
```

This will:
1. Recreate the `open-cluster-management-global-set` namespace
2. Restore the global placement and bindings
3. Reinstall all addons using global placement strategy
4. Restore ManagedClusterAddOn resources in all cluster namespaces

## Verification

After running the workaround command, verify recovery:

1. Check that the namespace is recreated:
```shell
oc get ns open-cluster-management-global-set
```

2. Verify the global placement is restored:
```shell
oc get placement global -n open-cluster-management-global-set
```

3. Confirm ManagedClusterAddOn resources are restored:
```shell
oc get managedclusteraddon -A | grep managed-serviceaccount
```

4. Check addon status across clusters:
```shell
oc get managedclusteraddon managed-serviceaccount -n <cluster-name> -o yaml
```

The addons should be automatically reinstalled and return to a healthy state.
