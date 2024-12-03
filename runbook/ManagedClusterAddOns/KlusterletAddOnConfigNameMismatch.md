# The ACM addons are installed several hours after the cluster is imported

## Symptom

The ACM addOns(`application-manager`, `cert-policy-controller`, `config-policy-controller`,
`governance-policy-framework`, `search-collector`) managed by the `KlusterletAddonConfig` are installed in the managed
cluster after several hours of the cluster importion. You can check with command below:

```shell
oc get managedclusteraddon -n <cluster-name> -o <addon-name>
```

and no `application-manager`, `cert-policy-controller`, `config-policy-controller`, `governance-policy-framework`,
`search-collector` return.

We expect these addons should be installed in several minutes after the cluster is imported.

## Meaning

These ACM addons are not successfully installed in clusters.

## Impact

Once the issue happens, the addon does not function properly. For instance, all policy and application related resources
will not be processed.

## Diagnosis

At first, check if the `KlusterletAddonConfig` with name `<cluster-name>` exists.
Check the `KlusterletAddonConfig` on the hub cluster.

```shell
oc get klusterletaddonconfig -n <cluster-name> <cluster-name> -o yaml
```

If the yaml exists, check the `spec.<addon-name>.enabled` field, the value should be `true`.

```yaml
spec:
  applicationManager:
    enabled: true
```

If the yaml does not exists, check if there are other `KlusterletAddonConfig` resources in the cluster namespace.

```shell
oc get klusterletaddonconfig -n <cluster-name> -o yaml
```

Note the name of the `KlusterletAddonConfig` resource should be **the same as the cluster name**, if not, delete the `KlusterletAddonConfig` resource and recreate it with the correct name.

Next check if the `ManagedClusterAddon` resource existing in the cluster namespace.

```shell
oc get managedclusteraddon <addon-name> -n <cluster-name>
```
