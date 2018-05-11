# Route Rules Alpha 3

Configuration affecting traffic routing. Here are a few terms useful to define in the context of traffic routing.

*Service* a unit of application behavior bound to a unique name in a service registry. Services consist of multiple network *endpoints* implemented by workload instances running on pods, containers, VMs etc.

*Service versions (subsets)* - In a continuous deployment scenario, for a given service, there can be distinct subsets of instances running different variants of the application binary. These variants are not necessarily different API versions. They could be iterative changes to the same service, deployed in different environments (prod, staging, dev, etc.). Common scenarios where this occurs include A/B testing, canary rollouts, etc. The choice of a particular version can be decided based on various criterion (headers, url, etc.) and/or by weights assigned to each version. Each service has a default version consisting of all its instances.

*Source* - A downstream client calling a service.

*Host* - The address used by a client when attempting to connect to a service.

*Access model* - Applications address only the destination service (Host) without knowledge of individual service versions (subsets). The actual choice of the version is determined by the proxy/sidecar, enabling the application code to decouple itself from the evolution of dependent services.

## ConnectionPoolSettings

Connection pool settings for an upstream host. The settings apply to each individual host in the upstream service. See Envoy’s [circuit breaker](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/circuit_breaking) for more details. Connection pool settings can be applied at the TCP level as well as at HTTP level.

For example, the following rule sets a limit of 100 connections to redis service called myredissrv with a connect timeout of 30ms

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-redis
spec:
  name: myredissrv
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
```

| Field  | Type                                  | Description                                                |
| ------ | ------------------------------------- | ---------------------------------------------------------- |
| `tcp`  | `ConnectionPoolSettings.TCPSettings`  | Settings common to both HTTP and TCP upstream connections. |
| `http` | `ConnectionPoolSettings.HTTPSettings` | HTTP connection pool settings.                             |

## ConnectionPoolSettings.HTTPSettings

Settings applicable to HTTP1.1/HTTP2/GRPC connections.

| Field                      | Type    | Description                                                  |
| -------------------------- | ------- | ------------------------------------------------------------ |
| `http1MaxPendingRequests`  | `int32` | Maximum number of pending HTTP requests to a destination. Default 1024. |
| `http2MaxRequests`         | `int32` | Maximum number of requests to a backend. Default 1024.       |
| `maxRequestsPerConnection` | `int32` | Maximum number of requests per connection to a backend. Setting this parameter to 1 disables keep alive. |
| `maxRetries`               | `int32` | Maximum number of retries that can be outstanding to all hosts in a cluster at a given time. Defaults to 3. |

## ConnectionPoolSettings.TCPSettings

Settings common to both HTTP and TCP upstream connections.

| Field            | Type                       | Description                                                  |
| ---------------- | -------------------------- | ------------------------------------------------------------ |
| `maxConnections` | `int32`                    | Maximum number of HTTP1 /TCP connections to a destination host. |
| `connectTimeout` | `google.protobuf.Duration` | TCP connection timeout.                                      |

## CorsPolicy

Describes the Cross-Origin Resource Sharing (CORS) policy, for a given service. Refer to https://developer.mozilla.org/en-US/docs/Web/HTTP/Access*control*CORS for further details about cross origin resource sharing. For example, the following rule restricts cross origin requests to those originating from example.com domain using HTTP POST/GET, and sets the Access-Control-Allow-Credentials header to false. In addition, it only exposes X-Foo-bar header and sets an expiry period of 1 day.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        name: ratings
        subset: v1
    corsPolicy:
      allowOrigin:
      - example.com
      allowMethods:
      - POST
      - GET
      allowCredentials: false
      allowHeaders:
      - X-Foo-Bar
      maxAge: "1d"
```

| Field              | Type                        | Description                                                  |
| ------------------ | --------------------------- | ------------------------------------------------------------ |
| `allowOrigin`      | `string[]`                  | The list of origins that are allowed to perform CORS requests. The content will be serialized into the Access-Control-Allow-Origin header. Wildcard * will allow all origins. |
| `allowMethods`     | `string[]`                  | List of HTTP methods allowed to access the resource. The content will be serialized into the Access-Control-Allow-Methods header. |
| `allowHeaders`     | `string[]`                  | List of HTTP headers that can be used when requesting the resource. Serialized to Access-Control-Allow-Methods header. |
| `exposeHeaders`    | `string[]`                  | A white list of HTTP headers that the browsers are allowed to access. Serialized into Access-Control-Expose-Headers header. |
| `maxAge`           | `google.protobuf.Duration`  | Specifies how long the the results of a preflight request can be cached. Translates to the Access-Control-Max-Age header. |
| `allowCredentials` | `google.protobuf.BoolValue` | Indicates whether the caller is allowed to send the actual request (not the preflight) using credentials. Translates to Access-Control-Allow-Credentials header. |

## Destination

Destination indicates the network addressable service to which the request/connection will be sent after processing a routing rule. The destination.name should unambiguously refer to a service in the service registry. It can be a short name or a fully qualified domain name from the service registry, a resolvable DNS name, an IP address or a service name from the service registry and a subset name. The order of inference is as follows:

1. Service registry lookup. The entire name is looked up in the service registry. If the lookup succeeds, the search terminates. The requests will be routed to any instance of the service in the mesh. When the service name consists of a single word, the FQDN will be constructed in a platform specific manner. For example, in Kubernetes, the namespace associated with the routing rule will be used to identify the service as .. However, if the service name contains multiple words separated by a dot (e.g., reviews.prod), the name in its entirety would be looked up in the service registry.
2. Runtime DNS lookup by the proxy. If step 1 fails, and the name is not an IP address, it will be considered as a DNS name that is not in the service registry (e.g., wikipedia.org). The sidecar/gateway will resolve the DNS and load balance requests appropriately. See Envoy’s strict_dns for details.

The following example routes all traffic by default to pods of the reviews service with label “version: v1” (i.e., subset v1), and some to subset v2, in a kubernetes environment.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews # namespace is same as the client/caller's namespace
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        name: reviews
        subset: v2
  - route:
    - destination:
        name: reviews
        subset: v1
```

And the associated DestinationRule

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  name: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

The following VirtualService sets a timeout of 5s for all calls to productpage.prod service. Notice that there are no subsets defined in this rule. Istio will fetch all instances of productpage.prod service from the service registry and populate the sidecar’s load balancing pool.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-productpage-rule
spec:
  hosts:
  - productpage.prod # in kubernetes, this applies only to prod namespace
  http:
  - timeout: 5s
    route:
    - destination:
        name: productpage.prod
```

The following sets a timeout of 5s for all calls to the external service wikipedia.org, as there is no internal service of that name.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-wiki-rule
spec:
  hosts:
  - wikipedia.org
  http:
  - timeout: 5s
    route:
    - destination:
        name: wikipedia.org
```

| Field    | Type           | Description                                                  |
| -------- | -------------- | ------------------------------------------------------------ |
| `name`   | `string`       | REQUIRED. The name can be a short name or a fully qualified domain name from the service registry, a resolvable DNS name, or an IP address.If short names are used, the FQDN of the service will be resolved in a platform specific manner. For example in Kubernetes, when a route with a short name “reviews” in the destination in namespace “bookinfo” is applied, the final destination is resolved to reviews.bookinfo.svc.cluster.local. The sidecar will route to the IP addresses of the pods constituting the service. However, if the lookup fails, “reviews” is treated as an external service, such that the sidecar will dynamically resolve the DNS of the service name and route the request to the IP addresses returned by the DNS. |
| `subset` | `string`       | The name of a subset within the service. Applicable only to services within the mesh. The subset must be defined in a corresponding DestinationRule. |
| `port`   | `PortSelector` | Specifies the port on the destination. Many services only expose a single port or label ports with the protocols they support, in these cases it is not required to explicitly select the port. Note that selection priority is to first match by name and then match by number.Names must comply with DNS label syntax (rfc1035) and therefore cannot collide with numbers. If there are multiple ports on a service with the same protocol the names should be of the form-. |

## DestinationRule

DestinationRule defines policies that apply to traffic intended for a service after routing has occurred. These rules specify configuration for load balancing, connection pool size from the sidecar, and outlier detection settings to detect and evict unhealthy hosts from the load balancing pool. For example, a simple load balancing policy for the ratings service would look as follows:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  name: ratings
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
```

Version specific policies can be specified by defining a named subset and overriding the settings specified at the service level. The following rule uses a round robin load balancing policy for all traffic going to a subset named testversion that is composed of endpoints (e.g., pods) with labels (version:v3).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  name: ratings
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

Note that policies specified for subsets will not take effect until a route rule explicitly sends traffic to this subset.

| Field           | Type            | Description                                                  |
| --------------- | --------------- | ------------------------------------------------------------ |
| `name`          | `string`        | REQUIRED. The destination address for traffic captured by this rule. Could be a DNS name with wildcard prefix or a CIDR prefix. Depending on the platform, short-names can also be used instead of a FQDN (i.e. has no dots in the name). In such a scenario, the FQDN of the host would be derived based on the underlying platform.For example on Kubernetes, when hosts contains a short name, Istio will interpret the short name based on the namespace of the rule. Thus, when a client applies a rule in the “default” namespace, containing a name “reviews”, Istio will setup routes to the “reviews.default.svc.cluster.local” service. However, if a different name such as “reviews.sales” is used, it would be treated as a FQDN during virtual host matching. In Consul, a plain service name would be resolved to the FQDN “reviews.service.consul”.Note that the hosts field applies to both HTTP and TCP services. Service inside the mesh, i.e. those found in the service registry, must always be referred to using their alphanumeric names. IP addresses or CIDR prefixes are allowed only for services defined via the Gateway. |
| `trafficPolicy` | `TrafficPolicy` | Traffic policies to apply (load balancing policy, connection pool sizes, outlier detection). |
| `subsets`       | `Subset[]`      | One or more named sets that represent individual versions of a service. Traffic policies can be overridden at subset level. |

## DestinationWeight

Each routing rule is associated with one or more service versions (see glossary in beginning of document). Weights associated with the version determine the proportion of traffic it receives. For example, the following rule will route 25% of traffic for the “reviews” service to instances with the “v2” tag and the remaining traffic (i.e., 75%) to “v1”.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        name: reviews
        subset: v2
      weight: 25
    - destination:
        name: reviews
        subset: v1
      weight: 75
```

And the associated DestinationRule

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  name: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

| Field         | Type          | Description                                                  |
| ------------- | ------------- | ------------------------------------------------------------ |
| `destination` | `Destination` | REQUIRED. Destination uniquely identifies the instances of a service to which the request/connection should be forwarded to. |
| `weight`      | `int32`       | REQUIRED. The proportion of traffic to be forwarded to the service version. (0-100). Sum of weights across destinations SHOULD BE == 100. If there is only destination in a rule, the weight value is assumed to be 100. |

## ExternalService

External service describes the endpoints, ports and protocols of a white-listed set of mesh-external domains and IP blocks that services in the mesh are allowed to access.

For example, the following external service configuration describes the set of services at https://example.com to be accessed internally over plaintext http (i.e. http://example.com:443), with the sidecar originating TLS.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ExternalService
metadata:
  name: external-svc-example
spec:
  hosts:
  - example.com
  ports:
  - number: 443
    name: example-http
    protocol: http # not HTTPS.
  discovery: DNS
```

and a destination rule to initiate TLS connections to the external service.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: tls-example
spec:
  name: example.com
  trafficPolicy:
    tls:
      mode: SIMPLE # initiates HTTPS when talking to example.com
```

The following specification specifies a static set of backend nodes for a MongoDB cluster behind a set of virtual IPs, and sets up a destination rule to initiate mTLS connections upstream.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ExternalService
metadata:
  name: external-svc-mongocluster
spec:
  hosts:
  - 192.192.192.192/24
  ports:
  - number: 27018
    name: mongodb
    protocol: mongo
  discovery: STATIC
  endpoints:
  - address: 2.2.2.2
  - address: 3.3.3.3
```

and the associated destination rule

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: mtls-mongocluster
spec:
  name: 192.192.192.192/24
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

The following example demonstrates the use of wildcards in the hosts. If the connection has to be routed to the IP address requested by the application (i.e. application resolves DNS and attempts to connect to a specific IP), the discovery mode must be set to “none”.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ExternalService
metadata:
  name: external-svc-wildcard-example
spec:
  hosts:
  - "*.bar.com"
  ports:
  - number: 80
    name: http
    protocol: http
  discovery: NONE
```

For HTTP based services, it is possible to create a virtual service backed by multiple DNS addressible endpoints. In such a scenario, the application can use the HTTP_PROXY environment variable to transparently reroute API calls for the virtual service to a chosen backend. For example, the following configuration creates a non-existent service called foo.bar.com backed by three domains: us.foo.bar.com:8443, uk.foo.bar.com:9443, and in.foo.bar.com:7443

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ExternalService
metadata:
  name: external-svc-dns
spec:
  hosts:
  - foo.bar.com
  ports:
  - number: 443
    name: https
    protocol: http
  discovery: DNS
  endpoints:
  - address: us.foo.bar.com
    ports:
      https: 8443
  - address: uk.foo.bar.com
    ports:
      https: 9443
  - address: in.foo.bar.com
    ports:
      https: 7443
```

and a destination rule to initiate TLS connections to the external service.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: tls-foobar
spec:
  name: foo.bar.com
  trafficPolicy:
    tls:
      mode: SIMPLE # initiates HTTPS
```

With HTTP_PROXY=http://localhost:443, calls from the application to http://foo.bar.com will be upgraded to HTTPS and load balanced across the three domains specified above. In other words, a call to http://foo.bar.com/baz would be translated to https://uk.foo.bar.com/baz.

NOTE: In the scenario above, the value of the HTTP Authority/host header associated with the outbound HTTP requests will be based on the endpoint’s DNS name, i.e. “:authority: uk.foo.bar.com”. Refer to Envoy’s auto*host*rewrite for further details. The automatic rewrite can be overridden using a host rewrite route rule.

| Field       | Type                         | Description                                                  |
| ----------- | ---------------------------- | ------------------------------------------------------------ |
| `hosts`     | `string[]`                   | REQUIRED. The hosts associated with the external service. Could be a DNS name with wildcard prefix or a CIDR prefix. Note that the hosts field applies to all protocols. DNS names in hosts will be ignored if the application accesses the service over non-HTTP protocols such as mongo/opaque TCP/even HTTPS. In such scenarios, the port on which the external service is being accessed must not be shared by any other service in the mesh. In other words, the sidecar will behave as a simple TCP proxy, forwarding incoming traffic on a specified port to the specified destination endpoint IP/host. |
| `ports`     | `Port[]`                     | REQUIRED. The ports associated with the external service.    |
| `discovery` | `ExternalService.Discovery`  | Service discovery mode for the hosts. If not set, Istio will attempt to infer the discovery mode based on the value of hosts and endpoints. |
| `endpoints` | `ExternalService.Endpoint[]` | One or more endpoints associated with the service. Endpoints must be accessible over the set of outPorts defined at the service level. |

## ExternalService.Discovery

Different ways of discovering the IP addresses associated with the service.

| Name     | Description                                                  |
| -------- | ------------------------------------------------------------ |
| `NONE`   | If set to “none”, the proxy will assume that incoming connections have already been resolved (to a specific destination IP address). Such connections are typically routed via the proxy using mechanisms such as IP table REDIRECT/ eBPF. After performing any routing related transformations, the proxy will forward the connection to the IP address to which the connection was bound. |
| `STATIC` | If set to “static”, the proxy will use the IP addresses specified in endpoints (See below) as the backing nodes associated with the external service. |
| `DNS`    | If set to “dns”, the proxy will attempt to resolve the DNS address during request processing. If no endpoints are specified, the proxy will resolve the DNS address specified in the hosts field, if wildcards are not used. If endpoints are specified, the DNS addresses specified in the endpoints will be resolved to determine the destination IP address. |

## ExternalService.Endpoint

Endpoint defines a network address (IP or hostname) associated with the external service.

| Field     | Type                  | Description                                                  |
| --------- | --------------------- | ------------------------------------------------------------ |
| `address` | `string`              | REQUIRED: Address associated with the network endpoint without the port ( IP or fully qualified domain name without wildcards). |
| `ports`   | `map<string, uint32>` | Set of ports associated with the endpoint. The ports must be associated with a port name that was declared as part of the service. |
| `labels`  | `map<string, string>` | One or more labels associated with the endpoint.             |

## Gateway

Gateway describes a load balancer operating at the edge of the mesh receiving incoming or outgoing HTTP/TCP connections. The specification describes a set of ports that should be exposed, the type of protocol to use, SNI configuration for the load balancer, etc.

For example, the following gateway spec sets up a proxy to act as a load balancer exposing port 80 and 9080 (http), 443 (https), and port 2379 (TCP) for ingress. The gateway will be applied to the proxy running on a pod with labels “app: my-gateway-controller”. While Istio will configure the proxy to listen on these ports, it is the responsibility of the user to ensure that external traffic to these ports are allowed into the mesh.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    app: my-gatweway-controller
  servers:
  - port:
      number: 80
      name: http
    hosts:
    - uk.bookinfo.com
    - eu.bookinfo.com
    tls:
      httpsRedirect: true # sends 302 redirect for http requests
  - port:
      number: 443
      name: https
    hosts:
    - uk.bookinfo.com
    - eu.bookinfo.com
    tls:
      mode: simple #enables HTTPS on this port
      serverCertificate: /etc/certs/servercert.pem
      privateKey: /etc/certs/privatekey.pem
  - port:
      number: 9080
      name: http-wildcard
    # no hosts implies wildcard match
  - port:
      number: 2379 #to expose internal service via external port 2379
      name: Mongo
      protocol: MONGO
```

The gateway specification above describes the L4-L6 properties of a load balancer. A VirtualService can then be bound to a gateway to control the forwarding of traffic arriving at a particular host or gateway port.

For example, the following VirtualService splits traffic for https://uk.bookinfo.com/reviews, https://eu.bookinfo.com/reviews, http://uk.bookinfo.com:9080/reviews, http://eu.bookinfo.com:9080/reviews into two versions (prod and qa) of an internal reviews service on port 9080. In addition, requests containing the cookie user: dev-123 will be sent to special port 7777 in the qa version. The same rule is also applicable inside the mesh for requests to the reviews.prod service. This rule is applicable across ports 443, 9080. Note that http://uk.bookinfo.com gets redirected to https://uk.bookinfo.com (i.e. 80 redirects to 443).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-rule
spec:
  hosts:
  - reviews.prod
  - uk.bookinfo.com
  - eu.bookinfo.com
  gateways:
  - my-gateway
  - mesh # applies to all the sidecars in the mesh
  http:
  - match:
    - headers:
        cookie:
          user: dev-123
    route:
    - destination:
        port:
          number: 7777
        name: reviews.qa
  - match:
      uri:
        prefix: /reviews/
    route:
    - destination:
        port:
          number: 9080 # can be omitted if its the only port for reviews
        name: reviews.prod
      weight: 80
    - destination:
        name: reviews.qa
      weight: 20
```

The following VirtualService forwards traffic arriving at (external) port 2379 from 172.17.16.0/24 subnet to internal Mongo server on port 5555. This rule is not applicable internally in the mesh as the gateway list omits the reserved name “mesh”.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-Mongo
spec:
  hosts:
  - Mongosvr #name of Mongo service
  gateways:
  - my-gateway
  tcp:
  - match:
    - port:
        number: 2379
      sourceSubnet: "172.17.16.0/24"
    route:
    - destination:
        name: mongo.prod
```

| Field      | Type                  | Description                                                  |
| ---------- | --------------------- | ------------------------------------------------------------ |
| `servers`  | `Server[]`            | REQUIRED: A list of server specifications.                   |
| `selector` | `map<string, string>` | One or more labels that indicate a specific set of pods/VMs on which this gateway configuration should be applied. If no selectors are provided, the gateway will be implemented by the default istio-ingress controller. |

## HTTPFaultInjection.Abort

Abort specification is used to prematurely abort a request with a pre-specified error code. The following example will return an HTTP 400 error code for 10% of the requests to the “ratings” service “v1”.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        name: ratings
        subset: v1
    fault:
      abort:
        percent: 10
        httpStatus: 400
```

The *httpStatus* field is used to indicate the HTTP status code to return to the caller. The optional *percent* field, a value between 0 and 100, is used to only abort a certain percentage of requests. If not specified, all requests are aborted.

| Field        | Type             | Description                                                  |
| ------------ | ---------------- | ------------------------------------------------------------ |
| `percent`    | `int32`          | Percentage of requests to be aborted with the error code provided (0-100). |
| `httpStatus` | `int32 (oneof)`  | REQUIRED. HTTP status code to use to abort the Http request. |
| `grpcStatus` | `string (oneof)` | (– NOT IMPLEMENTED –)                                        |
| `http2Error` | `string (oneof)` | (– NOT IMPLEMENTED –)                                        |

## HTTPFaultInjection.Delay

Delay specification is used to inject latency into the request forwarding path. The following example will introduce a 5 second delay in 10% of the requests to the “v1” version of the “reviews” service from all pods with label env: prod

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  http:
  - match:
    - sourceLabels:
        env: prod
    route:
    - destination:
        name: reviews
        subset: v1
    fault:
      delay:
        percent: 10
        fixedDelay: 5s
```

The *fixedDelay* field is used to indicate the amount of delay in seconds. An optional *percent* field, a value between 0 and 100, can be used to only delay a certain percentage of requests. If left unspecified, all request will be delayed.

| Field              | Type                               | Description                                                  |
| ------------------ | ---------------------------------- | ------------------------------------------------------------ |
| `percent`          | `int32`                            | Percentage of requests on which the delay will be injected (0-100). |
| `fixedDelay`       | `google.protobuf.Duration (oneof)` | REQUIRED. Add a fixed delay before forwarding the request. Format: 1h/1m/1s/1ms. MUST be >=1ms. |
| `exponentialDelay` | `google.protobuf.Duration (oneof)` | (– Add a delay (based on an exponential function) before forwarding the request. mean delay needed to derive the exponential delay values –) |

## HTTPMatchRequest

HttpMatchRequest specifies a set of criterion to be met in order for the rule to be applied to the HTTP request. For example, the following restricts the rule to match only requests where the URL path starts with /ratings/v2/ and the request contains a “cookie” with value “user=jason”.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        cookie:
          regex: "^(.*?;)?(user=jason)(;.*)?"
        uri:
          prefix: "/ratings/v2/"
    route: 
    - destination:
        name: ratings
```

HTTPMatchRequest CANNOT be empty.

| Field          | Type                       | Description                                                  |
| -------------- | -------------------------- | ------------------------------------------------------------ |
| `uri`          | `StringMatch`              | URI to match values are case-sensitive and formatted as follows:*exact: “value”* or just *“value”* for exact string match*prefix: “value”* for prefix-based match*regex: “value”* for ECMAscript style regex-based match |
| `scheme`       | `StringMatch`              | URI Scheme values are case-sensitive and formatted as follows:*exact: “value”* or just *“value”* for exact string match*prefix: “value”* for prefix-based match*regex: “value”* for ECMAscript style regex-based match |
| `method`       | `StringMatch`              | HTTP Method values are case-sensitive and formatted as follows:*exact: “value”* or just *“value”* for exact string match*prefix: “value”* for prefix-based match*regex: “value”* for ECMAscript style regex-based match |
| `authority`    | `StringMatch`              | HTTP Authority values are case-sensitive and formatted as follows:*exact: “value”* or just *“value”* for exact string match*prefix: “value”* for prefix-based match*regex: “value”* for ECMAscript style regex-based match |
| `headers`      | `map<string, StringMatch>` | The header keys must be lowercase and use hyphen as the separator, e.g. *x-request-id*.Header values are case-sensitive and formatted as follows:*exact: “value”* or just *“value”* for exact string match*prefix: “value”* for prefix-based match*regex: “value”* for ECMAscript style regex-based match*Note:* The keys *uri*, *scheme*, *method*, and *authority* will be ignored. |
| `port`         | `PortSelector`             | Specifies the ports on the host that is being addressed. Many services only expose a single port or label ports with the protocols they support, in these cases it is not required to explicitly select the port. Note that selection priority is to first match by name and then match by number.Names must comply with DNS label syntax (rfc1035) and therefore cannot collide with numbers. If there are multiple ports on a service with the same protocol the names should be of the form-. |
| `sourceLabels` | `map<string, string>`      | One or more labels that constrain the applicability of a rule to workloads with the given labels. If the VirtualService has a list of gateways specified at the top, it should include the reserved gateway “mesh” in order for this field to be applicable. |
| `gateways`     | `string[]`                 | Names of gateways where the rule should be applied to. Gateway names at the top of the VirtualService (if any) are overridden. The gateway match is independent of sourceLabels. |

## HTTPRedirect

HTTPRedirect can be used to send a 302 redirect response to the caller, where the Authority/Host and the URI in the response can be swapped with the specified values. For example, the following rule redirects requests for /v1/getProductRatings API on the ratings service to /v1/bookRatings provided by the bookratings service.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings
  http:
  - match:
    - uri:
        exact: /v1/getProductRatings
  redirect:
    uri: /v1/bookRatings
    authority: bookratings.default.svc.cluster.local
  ...
```

| Field       | Type     | Description                                                  |
| ----------- | -------- | ------------------------------------------------------------ |
| `uri`       | `string` | On a redirect, overwrite the Path portion of the URL with this value. Note that the entire path will be replaced, irrespective of the request URI being matched as an exact path or prefix. |
| `authority` | `string` | On a redirect, overwrite the Authority/Host portion of the URL with this value. |

## HTTPRetry

Describes the retry policy to use when a HTTP request fails. For example, the following rule sets the maximum number of retries to 3 when calling ratings:v1 service, with a 2s timeout per retry attempt.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        name: ratings
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
```

| Field           | Type                       | Description                                                  |
| --------------- | -------------------------- | ------------------------------------------------------------ |
| `attempts`      | `int32`                    | REQUIRED. Number of retries for a given request. The interval between retries will be determined automatically (25ms+). Actual number of retries attempted depends on the httpReqTimeout. |
| `perTryTimeout` | `google.protobuf.Duration` | Timeout per retry attempt for a given request. format: 1h/1m/1s/1ms. MUST BE >=1ms. |

## HTTPRewrite

HTTPRewrite can be used to rewrite specific parts of a HTTP request before forwarding the request to the destination. Rewrite primitive can be used only with the DestinationWeights. The following example demonstrates how to rewrite the URL prefix for api call (/ratings) to ratings service before making the actual API call.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings
  http:
  - match:
    - uri:
        prefix: /ratings
    rewrite:
      uri: /v1/bookRatings
    route:
    - destination:
        name: ratings
        subset: v1
```

| Field       | Type     | Description                                                  |
| ----------- | -------- | ------------------------------------------------------------ |
| `uri`       | `string` | rewrite the path (or the prefix) portion of the URI with this value. If the original URI was matched based on prefix, the value provided in this field will replace the corresponding matched prefix. |
| `authority` | `string` | rewrite the Authority/Host header with this value.           |

## HTTPRoute

Describes match conditions and actions for routing HTTP/1.1, HTTP2, and gRPC traffic. See VirtualService for usage examples.

| Field              | Type                       | Description                                                  |
| ------------------ | -------------------------- | ------------------------------------------------------------ |
| `match`            | `HTTPMatchRequest[]`       | Match conditions to be satisfied for the rule to be activated. All conditions inside a single match block have AND semantics, while the list of match blocks have OR semantics. The rule is matched if any one of the match blocks succeed. |
| `route`            | `DestinationWeight[]`      | A http rule can either redirect or forward (default) traffic. The forwarding target can be one of several versions of a service (see glossary in beginning of document). Weights associated with the service version determine the proportion of traffic it receives. |
| `redirect`         | `HTTPRedirect`             | A http rule can either redirect or forward (default) traffic. If traffic passthrough option is specified in the rule, route/redirect will be ignored. The redirect primitive can be used to send a HTTP 302 redirect to a different URI or Authority. |
| `rewrite`          | `HTTPRewrite`              | Rewrite HTTP URIs and Authority headers. Rewrite cannot be used with Redirect primitive. Rewrite will be performed before forwarding. |
| `websocketUpgrade` | `bool`                     | Indicates that a HTTP/1.1 client connection to this particular route should be allowed (and expected) to upgrade to a WebSocket connection. The default is false. Istio’s reference sidecar implementation (Envoy) expects the first request to this route to contain the WebSocket upgrade headers. Otherwise, the request will be rejected. Note that Websocket allows secondary protocol negotiation which may then be subject to further routing rules based on the protocol selected. |
| `timeout`          | `google.protobuf.Duration` | Timeout for HTTP requests.                                   |
| `retries`          | `HTTPRetry`                | Retry policy for HTTP requests.                              |
| `mirror`           | `Destination`              | Mirror HTTP traffic to a another destination in addition to forwarding the requests to the intended destination. Mirrored traffic is on a best effort basis where the sidecar/gateway will not wait for the mirrored cluster to respond before returning the response from the original destination. Statistics will be generated for the mirrored destination. |
| `corsPolicy`       | `CorsPolicy`               | Cross-Origin Resource Sharing policy (CORS). Refer to https://developer.mozilla.org/en-US/docs/Web/HTTP/Access*control*CORS for further details about cross origin resource sharing. |
| `appendHeaders`    | `map<string, string>`      | Additional HTTP headers to add before forwarding a request to the destination service. |

## L4MatchAttributes

L4 connection match attributes. Note that L4 connection matching support is incomplete.

| Field               | Type                  | Description                                                  |
| ------------------- | --------------------- | ------------------------------------------------------------ |
| `destinationSubnet` | `string`              | IPv4 or IPv6 ip address of destination with optional subnet. E.g., a.b.c.d/xx form or just a.b.c.d. This is only valid when the destination service has several IPs and the application explicitly specifies a particular IP. |
| `port`              | `PortSelector`        | Specifies the port on the host that is being addressed. Many services only expose a single port or label ports with the protocols they support, in these cases it is not required to explicitly select the port. Note that selection priority is to first match by name and then match by number.Names must comply with DNS label syntax (rfc1035) and therefore cannot collide with numbers. If there are multiple ports on a service with the same protocol the names should be of the form-. |
| `sourceSubnet`      | `string`              | IPv4 or IPv6 ip address of source with optional subnet. E.g., a.b.c.d/xx form or just a.b.c.d |
| `sourceLabels`      | `map<string, string>` | One or more labels that constrain the applicability of a rule to workloads with the given labels. If the VirtualService has a list of gateways specified at the top, it should include the reserved gateway “mesh” in order for this field to be applicable. |
| `gateways`          | `string[]`            | Names of gateways where the rule should be applied to. Gateway names at the top of the VirtualService (if any) are overridden. The gateway match is independent of sourceLabels. |

## LoadBalancerSettings

Load balancing policies to apply for a specific destination. See Envoy’s load balancing [documentation](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing.html) for more details.

For example, the following rule uses a round robin load balancing policy for all traffic going to the ratings service.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  name: ratings
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
```

The following example uses the consistent hashing based load balancer for the same ratings service using the Cookie header as the hash key.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  name: ratings
  trafficPolicy:
    loadBalancer:
      consistentHash:
        http_header: Cookie
```

| Field            | Type                                            | Description |
| ---------------- | ----------------------------------------------- | ----------- |
| `simple`         | `LoadBalancerSettings.SimpleLB (oneof)`         |             |
| `consistentHash` | `LoadBalancerSettings.ConsistentHashLB (oneof)` |             |

## LoadBalancerSettings.ConsistentHashLB

Consistent hashing (ketama hash) based load balancer for even load distribution/redistribution when the connection pool changes. This load balancing policy is applicable only for HTTP-based connections. A user specified HTTP header is used as the key with [xxHash](http://cyan4973.github.io/xxHash) hashing.

| Field             | Type     | Description                                                  |
| ----------------- | -------- | ------------------------------------------------------------ |
| `httpHeader`      | `string` | REQUIRED. The name of the HTTP request header that will be used to obtain the hash key. If the request header is not present, the load balancer will use a random number as the hash, effectively making the load balancing policy random. |
| `minimumRingSize` | `uint32` | The minimum number of virtual nodes to use for the hash ring. Defaults to 1024. Larger ring sizes result in more granular load distributions. If the number of hosts in the load balancing pool is larger than the ring size, each host will be assigned a single virtual node. |

## LoadBalancerSettings.SimpleLB

Standard load balancing algorithms that require no tuning.

| Name          | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| `ROUND_ROBIN` | Round Robin policy. Default                                  |
| `LEAST_CONN`  | The least request load balancer uses an O(1) algorithm which selects two random healthy hosts and picks the host which has fewer active requests. |
| `RANDOM`      | The random load balancer selects a random healthy host. The random load balancer generally performs better than round robin if no health checking policy is configured. |
| `PASSTHROUGH` | This option will forward the connection to the original IP address requested by the caller without doing any form of load balancing. This option must be used with care. It is meant for advanced use cases. Refer to Original Destination load balancer in Envoy for further details. |

## OutlierDetection

A Circuit breaker implementation that tracks the status of each individual host in the upstream service. While currently applicable to only HTTP services, future versions will support opaque TCP services as well. For HTTP services, hosts that continually return errors for API calls are ejected from the pool for a pre-defined period of time. See Envoy’s [outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/outlier) for more details.

The following rule sets a connection pool size of 100 connections and 1000 concurrent HTTP2 requests, with no more than 10 req/connection to “reviews” service. In addition, it configures upstream hosts to be scanned every 5 mins, such that any host that fails 7 consecutive times with 5XX error code will be ejected for 15 minutes.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-cb-policy
spec:
  name: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    outlierDetection:
      http:
        consecutiveErrors: 7
        interval: 5m
        baseEjectionTime: 15m
```

| Field  | Type                            | Description                                  |
| ------ | ------------------------------- | -------------------------------------------- |
| `http` | `OutlierDetection.HTTPSettings` | Settings for HTTP1.1/HTTP2/GRPC connections. |

## OutlierDetection.HTTPSettings

Outlier detection settings for HTTP1.1/HTTP2/GRPC connections.

| Field                | Type                       | Description                                                  |
| -------------------- | -------------------------- | ------------------------------------------------------------ |
| `consecutiveErrors`  | `int32`                    | Number of 5XX errors before a host is ejected from the connection pool. Defaults to 5. |
| `interval`           | `google.protobuf.Duration` | Time interval between ejection sweep analysis. format: 1h/1m/1s/1ms. MUST BE >=1ms. Default is 10s. |
| `baseEjectionTime`   | `google.protobuf.Duration` | Minimum ejection duration. A host will remain ejected for a period equal to the product of minimum ejection duration and the number of times the host has been ejected. This technique allows the system to automatically increase the ejection period for unhealthy upstream servers. format: 1h/1m/1s/1ms. MUST BE >=1ms. Default is 30s. |
| `maxEjectionPercent` | `int32`                    | Maximum % of hosts in the load balancing pool for the upstream service that can be ejected. Defaults to 10%. |

## Port

Port describes the properties of a specific port of a service.

| Field      | Type     | Description                                                  |
| ---------- | -------- | ------------------------------------------------------------ |
| `number`   | `uint32` | REQUIRED: A valid non-negative integer port number.          |
| `protocol` | `string` | The protocol exposed on the port. MUST BE one of HTTP\|HTTPS\|GRPC\|HTTP2\|MONGO\|TCP. |
| `name`     | `string` | Label assigned to the port.                                  |

## PortSelector

PortSelector specifies the name or number of a port to be used for matching or selection for final routing.

| Field    | Type             | Description       |
| -------- | ---------------- | ----------------- |
| `number` | `uint32 (oneof)` | Valid port number |
| `name`   | `string (oneof)` | Port name         |

## Server

Server describes the properties of the proxy on a given load balancer port. For example,

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-ingress
spec:
  selector:
    app: my-ingress-controller
  servers:
  - port:
      number: 80
      protocol: HTTP2
```

Another example

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-tcp-ingress
spec:
  selector:
    app: my-tcp-ingress-controller
  servers:
  - port:
      number: 27018
      protocol: MONGO
```

The following is an example of TLS configuration for port 443

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-tls-ingress
spec:
  selector:
    app: my-tls-ingress-controller
  servers:
  - port:
      number: 443
      protocol: HTTP
    tls:
      mode: simple
      serverCertificate: /etc/certs/server.pem
      privateKey: /etc/certs/privatekey.pem
```

| Field   | Type                | Description                                                  |
| ------- | ------------------- | ------------------------------------------------------------ |
| `port`  | `Port`              | REQUIRED: The Port on which the proxy should listen for incoming connections |
| `hosts` | `string[]`          | A list of hosts exposed by this gateway. While typically applicable to HTTP services, it can also be used for TCP services using TLS with SNI. Standard DNS wildcard prefix syntax is permitted.A VirtualService that is bound to a gateway must having a matching host in its default destination. Specifically one of the VirtualService destination hosts is a strict suffix of a gateway host or a gateway host is a suffix of one of the VirtualService hosts. |
| `tls`   | `Server.TLSOptions` | Set of TLS related options that govern the server’s behavior. Use these options to control if all http requests should be redirected to https, and the TLS modes to use. |

## Server.TLSOptions

| Field               | Type                        | Description                                                  |
| ------------------- | --------------------------- | ------------------------------------------------------------ |
| `httpsRedirect`     | `bool`                      | If set to true, the load balancer will send a 302 redirect for all http connections, asking the clients to use HTTPS. |
| `mode`              | `Server.TLSOptions.TLSmode` | Optional: Indicates whether connections to this port should be secured using TLS. The value of this field determines how TLS is enforced. |
| `serverCertificate` | `string`                    | REQUIRED if mode is “simple” or “mutual”. The path to the file holding the server-side TLS certificate to use. |
| `privateKey`        | `string`                    | REQUIRED if mode is “simple” or “mutual”. The path to the file holding the server’s private key. |
| `caCertificates`    | `string`                    | REQUIRED if mode is “mutual”. The path to a file containing certificate authority certificates to use in verifying a presented client side certificate. |
| `subjectAltNames`   | `string[]`                  | A list of alternate names to verify the subject identity in the certificate presented by the client. |

## Server.TLSOptions.TLSmode

TLS modes enforced by the proxy

| Name          | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| `PASSTHROUGH` | If set to “passthrough”, the proxy will forward the connection to the upstream server selected based on the SNI string presented by the client. |
| `SIMPLE`      | If set to “simple”, the proxy will secure connections with standard TLS semantics. |
| `MUTUAL`      | If set to “mutual”, the proxy will secure connections to the upstream using mutual TLS by presenting client certificates for authentication. |

## StringMatch

Describes how to match a given string in HTTP headers. Match is case-sensitive.

| Field    | Type             | Description                        |
| -------- | ---------------- | ---------------------------------- |
| `exact`  | `string (oneof)` | exact string match                 |
| `prefix` | `string (oneof)` | prefix-based match                 |
| `regex`  | `string (oneof)` | ECMAscript style regex-based match |

## Subset

A subset of endpoints of a service. Subsets can be used for scenarios like A/B testing, or routing to a specific version of a service. Refer to VirtualService documentation for examples of using subsets in these scenarios. In addition, traffic policies defined at the service-level can be overridden at a subset-level. The following rule uses a round robin load balancing policy for all traffic going to a subset named testversion that is composed of endpoints (e.g., pods) with labels (version:v3).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  name: ratings
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

Note that policies specified for subsets will not take effect until a route rule explicitly sends traffic to this subset.

| Field           | Type                  | Description                                                  |
| --------------- | --------------------- | ------------------------------------------------------------ |
| `name`          | `string`              | REQUIRED. name of the subset. The service name and the subset name can be used for traffic splitting in a route rule. |
| `labels`        | `map<string, string>` | REQUIRED. Labels apply a filter over the endpoints of a service in the service registry. See route rules for examples of usage. |
| `trafficPolicy` | `TrafficPolicy`       | Traffic policies that apply to this subset. Subsets inherit the traffic policies specified at the DestinationRule level. Settings specified at the subset level will override the corresponding settings specified at the DestinationRule level. |

## TCPRoute

Describes match conditions and actions for routing TCP traffic. The following routing rule forwards traffic arriving at port 2379 named Mongo from 172.17.16.* subnet to another Mongo server on port 5555.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-Mongo
spec:
  hosts:
  - myMongosrv 
  tcp:
  - match:
    - port:
        name: Mongo # only applies to ports named Mongo
      sourceSubnet: "172.17.16.0/24"
    route:
    - destination:
        name: mongo.prod
```

| Field   | Type                  | Description                                                  |
| ------- | --------------------- | ------------------------------------------------------------ |
| `match` | `L4MatchAttributes[]` | Match conditions to be satisfied for the rule to be activated. All conditions inside a single match block have AND semantics, while the list of match blocks have OR semantics. The rule is matched if any one of the match blocks succeed. |
| `route` | `DestinationWeight[]` | The destination to which the connection should be forwarded to. Currently, only one destination is allowed for TCP services. When TCP weighted routing support is introduced in Envoy, multiple destinations with weights can be specified. |

## TLSSettings

SSL/TLS related settings for upstream connections. See Envoy’s [TLS context](https://www.envoyproxy.io/docs/envoy/latest/api-v1/cluster_manager/cluster_ssl.html#config-cluster-manager-cluster-ssl) for more details. These settings are common to both HTTP and TCP upstreams.

For example, the following rule configures a client to use mutual TLS for connections to upstream database cluster.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: db-mtls
spec:
  name: mydbserver
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

The following rule configures a client to use TLS when talking to a foreign service whose domain matches *.foo.com.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: tls-foo
spec:
  name: "*.foo.com"
  trafficPolicy:
    tls:
      mode: SIMPLE
```

| Field               | Type                  | Description                                                  |
| ------------------- | --------------------- | ------------------------------------------------------------ |
| `mode`              | `TLSSettings.TLSmode` | REQUIRED: Indicates whether connections to this port should be secured using TLS. The value of this field determines how TLS is enforced. |
| `clientCertificate` | `string`              | REQUIRED if mode is “mutual”. The path to the file holding the client-side TLS certificate to use. |
| `privateKey`        | `string`              | REQUIRED if mode is “mutual”. The path to the file holding the client’s private key. |
| `caCertificates`    | `string`              | OPTIONAL: The path to the file containing certificate authority certificates to use in verifying a presented server certificate. If omitted, the proxy will not verify the server’s certificate. |
| `subjectAltNames`   | `string[]`            | A list of alternate names to verify the subject identity in the certificate. If specified, the proxy will verify that the server certificate’s subject alt name matches one of the specified values. |
| `sni`               | `string`              | SNI string to present to the server during TLS handshake.    |

## TLSSettings.TLSmode

TLS connection mode

| Name      | Description                                                  |
| --------- | ------------------------------------------------------------ |
| `DISABLE` | If set to “disable”, the proxy will use not setup a TLS connection to the upstream server. |
| `SIMPLE`  | If set to “simple”, the proxy will originate a TLS connection to the upstream server. |
| `MUTUAL`  | If set to “mutual”, the proxy will secure connections to the upstream using mutual TLS by presenting client certificates for authentication. |

## TrafficPolicy

Traffic policies to apply for a specific destination. See DestinationRule for examples.

| Field              | Type                     | Description                                                  |
| ------------------ | ------------------------ | ------------------------------------------------------------ |
| `loadBalancer`     | `LoadBalancerSettings`   | Settings controlling the load balancer algorithms.           |
| `connectionPool`   | `ConnectionPoolSettings` | Settings controlling the volume of connections to an upstream service |
| `outlierDetection` | `OutlierDetection`       | Settings controlling eviction of unhealthy hosts from the load balancing pool |
| `tls`              | `TLSSettings`            | TLS related settings for connections to the upstream service. |

## VirtualService

A VirtualService defines a set of traffic routing rules to apply when a host is addressed. Each routing rule defines matching criteria for traffic of a specific protocol. If the traffic is matched, then it is sent to a named destination service (or subset/version of it) defined in the registry.

The source of traffic can also be matched in a routing rule. This allows routing to be customized for specific client contexts.

The following example routes all HTTP traffic by default to pods of the reviews service with label “version: v1”. In addition, HTTP requests containing /wpcatalog/, /consumercatalog/ url prefixes will be rewritten to /newcatalog and sent to pods with label “version: v2”. The rules will be applied at the gateway named “bookinfo” as well as at all the sidecars in the mesh (indicated by the reserved gateway name “mesh”).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  gateways: # if omitted, defaults to "mesh"
  - bookinfo
  - mesh
  http:
  - match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        name: reviews
        subset: v2
  - route:
    - destination:
        name: reviews
        subset: v1
```

A subset/version of a route destination is identified with a reference to a named service subset which must be declared in a corresponding DestinationRule.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  name: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

A host name can be defined by only one VirtualService. A single VirtualService can be used to describe traffic properties for multiple HTTP and TCP ports.

| Field      | Type          | Description                                                  |
| ---------- | ------------- | ------------------------------------------------------------ |
| `hosts`    | `string[]`    | REQUIRED. The destination address for traffic captured by this virtual service. Could be a DNS name with wildcard prefix or a CIDR prefix. Depending on the platform, short-names can also be used instead of a FQDN (i.e. has no dots in the name). In such a scenario, the FQDN of the host would be derived based on the underlying platform.For example on Kubernetes, when hosts contains a short name, Istio will interpret the short name based on the namespace of the rule. Thus, when a client namespace applies a rule in the “default” namespace containing a name “reviews, Istio will setup routes to the “reviews.default.svc.cluster.local” service. However, if a different name such as “reviews.sales.svc.cluster.local” is used, it would be treated as a FQDN during virtual host matching. In Consul, a plain service name would be resolved to the FQDN “reviews.service.consul”.Note that the hosts field applies to both HTTP and TCP services. Service inside the mesh, i.e., those found in the service registry, must always be referred to using their alphanumeric names. IP addresses or CIDR prefixes are allowed only for services defined via the Gateway. |
| `gateways` | `string[]`    | The names of gateways and sidecars that should apply these routes. A single VirtualService is used for sidecars inside the mesh as well as for one or more gateways. The selection condition imposed by this field can be overridden using the source field in the match conditions of HTTP/TCP routes. The reserved word “mesh” is used to imply all the sidecars in the mesh. When this field is omitted, the default gateway (“mesh”) will be used, which would apply the rule to all sidecars in the mesh. If a list of gateway names is provided, the rules will apply only to the gateways. To apply the rules to both gateways and sidecars, specify “mesh” as one of the gateway names. |
| `http`     | `HTTPRoute[]` | An ordered list of route rules for HTTP traffic. The first rule matching an incoming request is used. |
| `tcp`      | `TCPRoute[]`  | An ordered list of route rules for TCP traffic. The first rule matching an incoming request is used. |