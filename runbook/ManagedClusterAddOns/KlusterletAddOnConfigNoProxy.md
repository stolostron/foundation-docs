# The ACM addon agents crashed by misconfigured proxy

## Symptom

A cluster is imported to ACM hub, the addOns(`application-manager`, `cert-policy-controller`, `config-policy-controller`,
`governance-policy-framework`, `search-collector`) managed by the `KlusterletAddonConfig` are installed in the managed
cluster, but addon status `Available` is `False`. And the addon agents is `CrashLoopBackOff`, some connecting errors in
the addon agent logs, you can check with command below:

```shell
oc get pod -n open-cluster-management-agent-addon
oc logs -n open-cluster-management-agent-addon <addon-agent-name>
```

The logs output contains:

```log
failed to get server groups: Get \"https://172.30.0.1:443/api\": context deadline exceeded
```

or

```log
proxyconnect tcp: dial tcp: lookup xxxxx on 172.30.0.10:53: no such host
```

We expect these addons should be in `Available` statue.

## Meaning

These ACM addons are installed with proxy configured, which results in the addon agents can not connect to the managed
cluster apiserver.

## Impact

Once the issue happens, the addon does not function properly. For instance, all policy and application related resources
will not be processed.

## Diagnosis

At first, check if the `KlusterletAddonConfig` with name `<cluster-name>` exists.
Check the `KlusterletAddonConfig` on the hub cluster.

```shell
oc get klusterletaddonconfig -n <cluster-name> <cluster-name> -o yaml
```

If the yaml exists, check if the `spec.proxyConfig.proxyConfig` is configured:

```yaml
  proxyConfig:
    httpProxy: http://proxytest:8080
    httpsProxy: http://proxytest:8080
    noProxy: 127.0.0.1,localhost,.cluster.local,.svc
```

Make sure the managed cluster service CIDR is in the `noProxy`, the address can be checked by command on the manged OCP cluster:

```shell
# OpenShift/ROSA:
oc get networks.config.openshift.io cluster -o jsonpath='{.spec.serviceNetwork}'
# Output: ["172.30.0.0/16"]
```

For example, ROSA HCP cluster has the default value `172.30.0.0/16`, this value should be added into the `noProxy` of
`KlusterletAddonConfig` on the hub cluster:

```yaml
  proxyConfig:
    httpProxy: http://proxytest:8080
    httpsProxy: http://proxytest:8080
    noProxy: 127.0.0.1,localhost,.cluster.local,.svc,172.30.0.0/16
```
