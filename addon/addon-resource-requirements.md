# Support customization of add-on agent resource requirements

## API
By using the `AddOnDeploymentConfig` API, user is able to specified the resource requirements of add-on agent per contianer. 
```go
type AddOnDeploymentConfigSpec struct {
	// ResourceRequirements specify the resources required by add-on agents.
	// If a container matches multiple ContainerResourceRequirements, the last matched configuration in the
	// array will take precedence.
	// +optional
	// +listType=map
	// +listMapKey=containerID
	ResourceRequirements []ContainerResourceRequirements `json:"resourceRequirements,omitempty"`
}

// ContainerResourceRequirements defines resources required by one or a group of containers.
type ContainerResourceRequirements struct {
	// ContainerID is a unique identifier for an agent container. It consists of three parts: resource types,
	// resource name, and container name, separated by ':'. The format follows
	// '{resource_types}:{resource_name}:{container_name}' where
	//   1). Supported resource types include deployments, daemonsets, statefulsets, replicasets, jobs,
	//     cronjobs and pods;
	//   2). Wildcards (*) can be used in any part to match multiple containers. For example, '*:*:*'
	//     matches all containers of the agent.
	// +required
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=`^(deployments|daemonsets|statefulsets|replicasets|jobs|cronjobs|pods|\*):.+:.+$`
	ContainerID string `json:"containerID"`

	// Compute resources required by matched containers.
	// +required
	// +kubebuilder:validation:Required
	Resources corev1.ResourceRequirements `json:"resources"`
}
```
Here is an example.
```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnDeploymentConfig
metadata:
  name: resource-requirements-example
  namespace: cluster1
spec:
  resourceRequirements:
  - containerID: *:*:*
    resources:
      limits:
        memory: 128Mi
      requests:
        memory: 16Mi
  - containerID: deployments:*:*
    resources:
      limits:
        memory: 256Mi
      requests:
        memory: 32Mi
  - containerID: deployments:deployment1:*
    resources:
      limits:
        memory: 512Mi
      requests:
        memory: 64Mi
  - containerID: deployments:deployment1:container1
    resources:
      limits:
        memory: 1024Mi
      requests:
        memory: 128Mi
```
If a contianer matches more than one `resourceRequirements`, the last matched one will take effect. For instance, the container `container1` in Deployment `deployment1` will adopt the 4th configuration, and the container `container1` in Deployment `deployment2` will adopt the second configuration.

## Development guide

### Addon built with helm charts
1. When implementing the add-on manager, append `addonfactory.ToAddOnResourceRequirementsValues` to the toValues function list of `addonfactory.GetAddOnDeploymentConfigValues()` to load the resource requirements settings from the AddOnDeploymentConfig API.
```go
	agentAddon, err := addonfactory.NewAgentAddonFactory(helloworld_helm.AddonName, helloworld_helm.FS, "manifests/charts/helloworld").
		WithGetValuesFuncs(
			addonfactory.GetAddOnDeploymentConfigValues(
				utils.NewAddOnDeploymentConfigGetter(addonClient),
				addonfactory.ToAddOnNodePlacementValues,
				addonfactory.ToAddOnProxyConfigValues,
				addonfactory.ToAddOnResourceRequirementsValues,
			),
            ... ...
		).
    ... ...
```

When rendering the helm chart, the requirements settings loaded from the above example `AddOnDeploymentConfig` resource can be accessed by using `.Values.global.resourceRequirements` with content below.
```yaml
global:
  resourceRequirements:
  - containerIDRegex: ^.+:.+:.+$
    resources:
      requests:
        memory: 16Mi
      limits:
        memory: 128Mi
  - containerIDRegex: ^deployments:.+:.+$
    resources:
      requests:
        memory: 32Mi
      limits:
        memory: 256Mi
  - containerIDRegex: ^deployments:deployment1:.+$
    resources:
      requests:
        memory: 64Mi
      limits:
        memory: 512Mi
  - containerIDRegex: ^deployments:deployment1:container1$
    resources:
      requests:
        memory: 128Mi
      limits:
        memory: 1024Mi
```

2. Set the default resource requirements in the `values.yaml` of the add-on chart.

An example looks like,
```yaml
global:
  resourceRequirements:
  - containerIDRegex: ^.+:.+:.+$
    resources:
      requests:
        memory: 32Mi
      limits:
        memory: 128Mi
```
Or you can set with empty value.
```yaml
global:
  resourceRequirements: []
```

3. Set the resource requirements per contianer in the add-on deployments, daemonsets and pods.
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: deployment1
  namespace: {{ .Release.Namespace }}
spec:
  template:
    spec:
      containers:
      - name: container1
        {{- $reverseResourceRequirements := reverse .Values.global.resourceRequirements }}
        {{- range $item := $reverseResourceRequirements }}
          {{- if regexMatch $item.containerIDRegex "deployments:deployment1:container1" }}
        resources:
            {{- toYaml $item.resources | nindent 10 }}
            {{- break -}}
          {{- end -}}
        {{- end }}
``` 

An example add-ons can be found here:
- https://github.com/open-cluster-management-io/addon-framework/blob/main/cmd/example/helloworld_helm/main.go
- https://github.com/open-cluster-management-io/addon-framework/tree/main/examples/helloworld_helm/manifests/charts/helloworld

### Addon built with raw manifests

1. When implementing the add-on manager, make sure `addonfactory.ToAddOnDeploymentConfigValues` is added into the toValues function list of `addonfactory.GetAddOnDeploymentConfigValues`.
```go
	agentAddon, err := addonfactory.NewAgentAddonFactory(helloworld.AddonName, helloworld.FS, "manifests/templates").
		WithGetValuesFuncs(
			addonfactory.GetAddOnDeploymentConfigValues(
				utils.NewAddOnDeploymentConfigGetter(addonClient),
				addonfactory.ToAddOnDeploymentConfigValues,
			),
            ... ...
		).
    ... ...
```
When rendering the manifests, the requirements settings loaded from the above example `AddOnDeploymentConfig` resource can be accessed with `.ResourceRequirements` with content below.
```yaml
ResourceRequirements:
  - containerIDRegex: '^.+:.+:.+$'
    resources:
      requests:
        memory: 16Mi
      limits:
        memory: 128Mi
  - containerIDRegex: '^deployments:.+:.+$'
    resources:
      requests:
        memory: 32Mi
      limits:
        memory: 256Mi
  - containerIDRegex: '^deployments:deployment1:.+$'
    resources:
      requests:
        memory: 64Mi
      limits:
        memory: 512Mi
  - containerIDRegex: '^deployments:deployment1:container1$'
    resources:
      requests:
        memory: 128Mi
      limits:
        memory: 1024Mi
```

2. Set the resource requirements per contianer in the add-on deployments, daemonsets and pods.
```yaml
apiVersion: apps/v1
metadata:
  name: deployment1
  namespace: {{ .AddonInstallNamespace }}
spec:
  template:
    spec:
      containers:
      - name: container1
{{- with .ResourceRequirements}}
    {{- $matchedIndex := -1 }}
    {{- range $index, $item := . }}
        {{- if regexMatch $item.ContainerIDRegex "deployments:deployment1:container1" }}
          {{- $matchedIndex = $index }}
        {{- end }}
    {{- end }}
    {{- if ne $matchedIndex -1 }}
        {{- $matched := index . $matchedIndex }}
        resources:
        {{- if $matched.Resources.Requests}}
          requests:
          {{- range $key, $value := $matched.Resources.Requests }}
            "{{ $key }}": "{{ $value }}"
          {{- end }}
        {{- end }}
        {{- if $matched.Resources.Limits}}
          limits:
          {{- range $key, $value := $matched.Resources.Limits }}
            "{{ $key }}": "{{ $value }}"
          {{- end }}
        {{- end }}
    {{- end }}
{{- end }}
```

An example add-ons can be found here:
- https://github.com/open-cluster-management-io/addon-framework/blob/main/cmd/example/helloworld/main.go
- https://github.com/open-cluster-management-io/addon-framework/tree/main/examples/helloworld/manifests/templates

### Addon defined with AddOnTemplate API
Currently, add-on defined with AddOnTemplate API does not support this feature.