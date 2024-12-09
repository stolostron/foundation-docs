# The `work-manager` addOn is not Available

## Symptom

The `work-manager` `ManagedClusterAddon` resource is created, but the condition `Available` is `False`. you can check
with command below:

```shell
oc get managedclusteraddon -n <cluster-name> work-manager -ojsonpath='{.status.conditions[?(@.type=="Available")].status}'
```

and it returns `False`.

## Meaning

The `work-manager` addon is not successfully installed in the managed cluster.

## Impact

Once the issue happens, the `work-manager` addon does not function properly. For instance, the URL of the managed
cluster or the `ClusterClaim` will not be populated to the `ManagedCluster` resource, the `ManagedClusterInfo` resource
is not updated, retriving logs for a specific pod in the managed cluster will fail either.

## Diagnosis

At first, check if the addon manager of the `work-manager` addon is started successfully on the hub cluster. Check the
`ocm-controller` logs with the command below, the logs should contain the message `Addon deploy is enabled, starting
addon manager`:

```shell
logs -n multicluster-engine <ocm-controller-xxx> | grep "Addon deploy is enabled, starting addon manager"
```

There are two pods of the `ocm-controller` in the `multicluster-engine` namespace, if both of them do not contain the
message, the addon manager is not started successfully. we can restart the `ocm-controller` pods with the command below:

```shell
oc delete pod -n multicluster-engine -l control-plane=ocm-controller
```

After the `ocm-controller` pods are restarted, check the logs again to see if the addon manager is started successfully
and the `work-manager` addon `Available` status is updated to `True`.
