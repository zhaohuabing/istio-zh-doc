# Service Mesh

## AuthenticationPolicy

AuthenticationPolicy defines authentication policy. It can be set for different scopes (mesh, service …), and the most narrow scope with non-INHERIT value will be used. Mesh policy cannot be INHERIT.

| Name         | Description                                                  |
| ------------ | ------------------------------------------------------------ |
| `NONE`       | Do not encrypt Envoy to Envoy traffic.                       |
| `MUTUAL_TLS` | Envoy to Envoy traffic is wrapped into mutual TLS connections. |
| `INHERIT`    | Use the policy defined by the parent scope. Should not be used for mesh policy. |

## MeshConfig

MeshConfig defines mesh-wide variables shared by all Envoy instances in the Istio service mesh.

| Field                   | Type                               | Description                                                  |
| ----------------------- | ---------------------------------- | ------------------------------------------------------------ |
| `mixerCheckServer`      | `string`                           | Address of the server that will be used by the proxies for policy check calls. By using different names for mixerCheckServer and mixerReportServer, it is possible to have one set of mixer servers handle policy check calls while another set of mixer servers handle telemetry calls.NOTE: Omitting mixerCheckServer while specifying mixerReportServer is equivalent to setting disablePolicyChecks to true. |
| `mixerReportServer`     | `string`                           | Address of the server that will be used by the proxies for policy report calls. |
| `disablePolicyChecks`   | `bool`                             | Disable policy checks by the mixer service. Default is false, i.e. mixer policy check is enabled by default. |
| `proxyListenPort`       | `int32`                            | Port on which Envoy should listen for incoming connections from other services. |
| `proxyHttpPort`         | `int32`                            | Port on which Envoy should listen for HTTP PROXY requests if set. |
| `connectTimeout`        | `google.protobuf.Duration`         | Connection timeout used by Envoy. (MUST BE >=1ms)            |
| `ingressClass`          | `string`                           | Class of ingress resources to be processed by Istio ingress controller. This corresponds to the value of “kubernetes.io/ingress.class” annotation. |
| `ingressService`        | `string`                           | Name of the kubernetes service used for the istio ingress controller. |
| `ingressControllerMode` | `MeshConfig.IngressControllerMode` | Defines whether to use Istio ingress controller for annotated or all ingress resources. |
| `authPolicy`            | `MeshConfig.AuthPolicy`            | Authentication policy defines the global switch to control authentication for Envoy-to-Envoy communication. Use authentication_policy instead. |
| `rdsRefreshDelay`       | `google.protobuf.Duration`         | Polling interval for RDS (MUST BE >=1ms)                     |
| `enableTracing`         | `bool`                             | Flag to control generation of trace spans and request IDs. Requires a trace span collector defined in the proxy configuration. |
| `accessLogFile`         | `string`                           | File address for the proxy access log (e.g. /dev/stdout). Empty value disables access logging. |
| `defaultConfig`         | `ProxyConfig`                      | Default proxy config used by the proxy injection mechanism operating in the mesh (e.g. Kubernetes admission controller) In case of Kubernetes, the proxy config is applied once during the injection process, and remain constant for the duration of the pod. The rest of the mesh config can be changed at runtime and config gets distributed dynamically. |
| `mtlsExcludedServices`  | `string[]`                         | List of remote services for which mTLS authentication should not be expected by Istio . Typically, these are control services (e.g kubernetes API server) that don’t have Istio sidecar to transparently terminate mTLS authentication. It has no effect if the authentication policy is already ‘NONE’. DO NOT use this setting for services that are managed by Istio (i.e. using Istio sidecar). Instead, use service-level annotations to overwrite the authentication policy. |
| `mixerAddress`          | `string`                           | DEPRECATED. Mixer address. This option will be removed soon. Please use mixer*check and mixer*report. |

## MeshConfig.AuthPolicy

TODO AuthPolicy needs to be removed and merged with AuthPolicy defined above

| Name         | Description                                                  |
| ------------ | ------------------------------------------------------------ |
| `NONE`       | Do not encrypt Envoy to Envoy traffic.                       |
| `MUTUAL_TLS` | Envoy to Envoy traffic is wrapped into mutual TLS connections. |

## MeshConfig.IngressControllerMode

| Name      | Description                                                  |
| --------- | ------------------------------------------------------------ |
| `OFF`     | Disables Istio ingress controller.                           |
| `DEFAULT` | Istio ingress controller will act on ingress resources that do not contain any annotation or whose annotations match the value specified in the ingress_class parameter described earlier. Use this mode if Istio ingress controller will be the default ingress controller for the entire kubernetes cluster. |
| `STRICT`  | Istio ingress controller will only act on ingress resources whose annotations match the value specified in the ingress_class parameter described earlier. Use this mode if Istio ingress controller will be a secondary ingress controller (e.g., in addition to a cloud-provided ingress controller). |

## ProxyConfig

ProxyConfig defines variables for individual Envoy instances.

| Field                        | Type                       | Description                                                  |
| ---------------------------- | -------------------------- | ------------------------------------------------------------ |
| `configPath`                 | `string`                   | Path to the generated configuration file directory. Proxy agent generates the actual configuration and stores it in this directory. |
| `binaryPath`                 | `string`                   | Path to the proxy binary                                     |
| `serviceCluster`             | `string`                   | Service cluster defines the name for the service_cluster that is shared by all Envoy instances. This setting corresponds to *–service-cluster* flag in Envoy. In a typical Envoy deployment, the *service-cluster* flag is used to identify the caller, for source-based routing scenarios.Since Istio does not assign a local service/service version to each Envoy instance, the name is same for all of them. However, the source/caller’s identity (e.g., IP address) is encoded in the *–service-node* flag when launching Envoy. When the RDS service receives API calls from Envoy, it uses the value of the *service-node* flag to compute routes that are relative to the service instances located at that IP address. |
| `drainDuration`              | `google.protobuf.Duration` | The time in seconds that Envoy will drain connections during a hot restart. MUST be >=1s (e.g., *1s/1m/1h*) |
| `parentShutdownDuration`     | `google.protobuf.Duration` | The time in seconds that Envoy will wait before shutting down the parent process during a hot restart. MUST be >=1s (e.g., *1s/1m/1h*). MUST BE greater than *drain*duration_ parameter. |
| `discoveryAddress`           | `string`                   | Address of the discovery service exposing xDS with mTLS connection. |
| `discoveryRefreshDelay`      | `google.protobuf.Duration` | Polling interval for service discovery (used by EDS, CDS, LDS, but not RDS). (MUST BE >=1ms) |
| `zipkinAddress`              | `string`                   | Address of the Zipkin service (e.g. *zipkin:9411*).          |
| `connectTimeout`             | `google.protobuf.Duration` | Connection timeout used by Envoy for supporting services. (MUST BE >=1ms) |
| `statsdUdpAddress`           | `string`                   | IP Address and Port of a statsd UDP listener (e.g. *10.75.241.127:9125*). |
| `proxyAdminPort`             | `int32`                    | Port on which Envoy should listen for administrative commands. |
| `availabilityZone`           | `string`                   | The availability zone where this Envoy instance is running. When running Envoy as a sidecar in Kubernetes, this flag must be one of the availability zones assigned to a node using failure-domain.beta.kubernetes.io/zone annotation. |
| `controlPlaneAuthPolicy`     | `AuthenticationPolicy`     | Authentication policy defines the global switch to control authentication for Envoy-to-Envoy communication for istio components Mixer and Pilot. |
| `customConfigFile`           | `string`                   | File path of custom proxy configuration, currently used by proxies in front of Mixer and Pilot. |
| `statNameLength`             | `int32`                    | Maximum length of name field in Envoy’s metrics. The length of the name field is determined by the length of a name field in a service and the set of labels that comprise a particular version of the service. The default value is set to 189 characters. Envoy’s internal metrics take up 67 characters, for a total of 256 character name per metric. Increase the value of this field if you find that the metrics from Envoys are truncated. |
| `concurrency`                | `int32`                    | The number of worker threads to run. Default value is number of cores on the machine. |
| `proxyBootstrapTemplatePath` | `string`                   | Path to the proxy bootstrap template file                    |