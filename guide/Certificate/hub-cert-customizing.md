# Customizing the hub cluster KubeAPIServer certificates

When connecting to the hub cluster's KubeAPIServer, the klusterlet on managed clusters uses a CA bundle to verify the hub's certificate. ACM determines this CA bundle based on the `hubKubeAPIServerConfig` in the KlusterletConfig and propagates it to all managed clusters.

To view the certificates in the currently used CA bundle on a managed cluster, run the command below on the hub cluster:

```bash
# Run on the hub cluster, replace <managed-cluster-name> with your cluster name
oc -n <managed-cluster-name> get secret <managed-cluster-name>-import -o jsonpath='{.data.import\.yaml}' | base64 -d | grep bootstrap-hub-kubeconfig -A 4 | grep "kubeconfig:" | awk '{ system("echo "$2" | base64 -d")}' | grep certificate-authority-data | awk '{system("echo "$2" | base64 -d")}'
```

**Note:**
- This command does not work for ManagedCluster with label `local-cluster:true` since it uses the internal endpoint to access the hub KubeAPIServer.
- For other clusters, if the output is empty, that indicates the system trust store is being used by the klusterlet.

Updating the hub cluster's KubeAPIServer certificate may cause managed clusters to go offline unless you follow the correct procedure. The procedure depends on your hub cluster KubeAPIServer verification strategy and what's changing in the certificate chain.

ACM uses ServerVerificationStrategy in the KlusterletConfig to control how managed clusters verify the hub cluster's KubeAPIServer certificate. There are three strategies: UseAutoDetectedCABundle, UseSystemTruststore, and UseCustomCABundles. For detailed information about each strategy and their behaviors, see [Setting the hub cluster KubeAPIServer verification strategy](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.15/html-single/clusters/index#set-hub-kube-api-server).

To check your current strategy, run:

```bash
oc get klusterletconfig global -o yaml
```

Look for `spec.hubKubeAPIServerConfig.serverVerificationStrategy`. If the klusterletconfig doesn't exist or this field is not set, UseAutoDetectedCABundle is in use.

## Certificate update scenarios

You might need to update the hub cluster certificate in the following situations:

### Scenario 1: Certificate renewal (same root and intermediate CA)

You're renewing an expiring certificate, but both the intermediate and root CA remain unchanged. This scenario has no risk of managed cluster disruption.

**All strategies:** Follow [Procedure A: Simple Certificate Update](#procedure-a-simple-certificate-update).

### Scenario 2: New intermediate CA (same root CA)

You're updating to a certificate with a new intermediate CA, but the root CA remains the same. The procedure depends on whether the root CA is already trusted by managed clusters.

**UseAutoDetectedCABundle strategy:**
- If the CA bundle currently used by klusterlet includes the root CA: Follow [Procedure A: Simple Certificate Update](#procedure-a-simple-certificate-update). ACM will auto-detect the new intermediate CA.
- If the root CA is not in the CA bundle currently used by klusterlet: Follow [Procedure B: Add Root CA and Update Certificate](#procedure-b-add-root-ca-and-update-certificate).

**UseSystemTruststore strategy:**
- Follow [Procedure A: Simple Certificate Update](#procedure-a-simple-certificate-update). The root CA is already in the system trust store.

**UseCustomCABundles strategy:**
- If the root CA is in your configured CA bundles: Follow [Procedure A: Simple Certificate Update](#procedure-a-simple-certificate-update).
- If the root CA is not in your configured CA bundles: Follow [Procedure B: Add Root CA and Update Certificate](#procedure-b-add-root-ca-and-update-certificate).

### Scenario 3: New root CA

You're updating to a certificate with a different root CA.

**Warning:** Do not update the KubeAPIServer certificate directly when the root CA changes, as this may cause managed clusters to go offline immediately.

**UseAutoDetectedCABundle strategy:**
- Follow [Procedure C: 3-Phase Root CA Update with Auto-Detection](#procedure-c-3-phase-root-ca-update-with-auto-detection).

**UseSystemTruststore strategy:**
- If the new root CA is publicly trusted and present in all managed cluster system trust stores: Follow [Procedure A: Simple Certificate Update](#procedure-a-simple-certificate-update).
- If the new root CA is not publicly trusted: Switch to UseCustomCABundles strategy first by specifying the current CA bundle in a ConfigMap, then follow [Procedure D: 3-Phase Root CA Update with Custom CA Bundles](#procedure-d-3-phase-root-ca-update-with-custom-ca-bundles).

**UseCustomCABundles strategy:**
- Follow [Procedure D: 3-Phase Root CA Update with Custom CA Bundles](#procedure-d-3-phase-root-ca-update-with-custom-ca-bundles).

## Procedures

### Procedure A: Simple Certificate Update

This procedure directly updates the KubeAPIServer certificate when managed clusters already trust the CA chain.

1. Follow the [OpenShift documentation for configuring API server certificates](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/configuring-certificates#api-server-certificates) to update the KubeAPIServer certificate. Ensure the certificate file includes the complete certificate chain (leaf certificate, intermediate CA certificates, and root CA certificate) in the correct order.

2. Verify that managed clusters remain available and stay available for at least 15 minutes:

   ```bash
   oc get managedclusters -w
   # All should show AVAILABLE=True and remain stable
   ```

### Procedure B: Add Root CA and Update Certificate

This procedure adds a root CA to the klusterlet CA bundles and then updates the certificate. It works for both UseAutoDetectedCABundle and UseCustomCABundles strategies.

1. Extract the root CA certificate and save as `root-ca-bundle.crt`.

2. Create a ConfigMap with the root CA bundle:

   ```bash
   oc create configmap root-ca-bundle \
     --from-file=ca.crt=/path/to/root-ca-bundle.crt \
     -n multicluster-engine
   ```

3. Add the root CA to trustedCABundles:

   a. Edit the global KlusterletConfig:

   ```bash
   oc edit klusterletconfig global
   ```

   b. Add the root CA bundle to `trustedCABundles`:

   ```yaml
   spec:
     hubKubeAPIServerConfig:
       trustedCABundles:
       - name: root-ca
         caBundle:
           name: root-ca-bundle
           namespace: multicluster-engine
   ```

4. Wait for the configuration to propagate to all managed clusters (usually within 5 minutes).

5. Follow [Procedure A: Simple Certificate Update](#procedure-a-simple-certificate-update) to update the certificate and verify managed clusters remain available.

### Procedure C: 3-Phase Root CA Update with Auto-Detection

This procedure updates to a new root CA while using UseAutoDetectedCABundle strategy. It temporarily adds the new CA to trustedCABundles, updates the certificate, then removes the temporary configuration.

**Phase 1: Add new CA to trustedCABundles**

1. Prepare the new CA bundle by extracting the root and intermediate CA certificates from your new certificate and save as `new-ca-bundle.crt`. Do not include the leaf certificate.

2. Create a ConfigMap with the new CA bundle:

   ```bash
   oc create configmap new-root-ca-bundle \
     --from-file=ca.crt=/path/to/new-ca-bundle.crt \
     -n multicluster-engine
   ```

3. Add the new CA bundle to trustedCABundles:

   a. Edit the global KlusterletConfig:

   ```bash
   oc edit klusterletconfig global
   ```

   b. Add the new CA bundle to `trustedCABundles`:

   ```yaml
   spec:
     hubKubeAPIServerConfig:
       trustedCABundles:
       - name: new-root-ca
         caBundle:
           name: new-root-ca-bundle
           namespace: multicluster-engine
   ```

   Managed clusters will now trust both the auto-detected old CA bundle and the new CA bundle.

4. Wait for the configuration to propagate to all managed clusters (usually within 5 minutes).

**Phase 2: Update the certificate**

5. Follow [Procedure A: Simple Certificate Update](#procedure-a-simple-certificate-update) to update the certificate and verify managed clusters remain available.

**Phase 3: Remove the temporary CA configuration**

6. Remove the additional CA bundle configuration:

   a. Edit the global KlusterletConfig:

   ```bash
   oc edit klusterletconfig global
   ```

   b. Remove the `trustedCABundles` section or the entire `hubKubeAPIServerConfig` section:

   ```yaml
   apiVersion: config.open-cluster-management.io/v1alpha1
   kind: KlusterletConfig
   metadata:
     name: global
   spec: {}
     # hubKubeAPIServerConfig section removed
   ```

7. Wait for the configuration to propagate to all managed clusters (usually within 5 minutes).

8. Verify that managed clusters remain available and stay available for at least 15 minutes:

   ```bash
   oc get managedclusters -w
   # All should show AVAILABLE=True and remain stable
   ```

9. Delete the temporary ConfigMap:

   ```bash
   oc delete configmap new-root-ca-bundle -n multicluster-engine
   ```

### Procedure D: 3-Phase Root CA Update with Custom CA Bundles

This procedure updates to a new root CA while using UseCustomCABundles strategy. It adds the new CA to trustedCABundles, updates the certificate, then removes the old CA.

**Phase 1: Add new CA to trustedCABundles**

1. Prepare the new CA bundle by extracting the root and intermediate CA certificates from your new certificate and save as `new-ca-bundle.crt`. Do not include the leaf certificate.

2. Create a ConfigMap with the new CA bundle:

   ```bash
   oc create configmap new-root-ca-bundle \
     --from-file=ca.crt=/path/to/new-ca-bundle.crt \
     -n multicluster-engine
   ```

3. Add the new CA bundle to trustedCABundles:

   a. Edit the global KlusterletConfig:

   ```bash
   oc edit klusterletconfig global
   ```

   b. Add the new CA bundle to the existing list (do not remove existing entries):

   ```yaml
   spec:
     hubKubeAPIServerConfig:
       serverVerificationStrategy: UseCustomCABundles
       trustedCABundles:
       # Keep existing entries
       - name: existing-ca
         caBundle:
           name: existing-ca-configmap
           namespace: multicluster-engine
       # Add new entry
       - name: new-root-ca
         caBundle:
           name: new-root-ca-bundle
           namespace: multicluster-engine
   ```

   Managed clusters will now trust both the old and new CA bundles.

4. Wait for the configuration to propagate to all managed clusters (usually within 5 minutes).

**Phase 2: Update the certificate**

5. Follow [Procedure A: Simple Certificate Update](#procedure-a-simple-certificate-update) to update the certificate and verify managed clusters remain available.

**Phase 3: Remove the old CA**

6. Remove the old CA bundle from trustedCABundles:

   a. Edit the global KlusterletConfig:

   ```bash
   oc edit klusterletconfig global
   ```

   b. Remove the old CA bundle entry, keeping only the new one:

   ```yaml
   spec:
     hubKubeAPIServerConfig:
       serverVerificationStrategy: UseCustomCABundles
       trustedCABundles:
       # Remove old entries, keep only what's needed
       - name: new-root-ca
         caBundle:
           name: new-root-ca-bundle
           namespace: multicluster-engine
   ```

7. Wait for the configuration to propagate to all managed clusters (usually within 5 minutes).

8. Verify that managed clusters remain available and stay available for at least 15 minutes:

   ```bash
   oc get managedclusters -w
   # All should show AVAILABLE=True and remain stable
   ```

9. Optionally delete the old ConfigMap if no longer needed:

   ```bash
   oc delete configmap existing-ca-configmap -n multicluster-engine
   ```
