# Route Rules Alpha 1

Configuration affecting traffic routing. Here are a few terms useful to define in the context of routing rules.

*Service* is a unit of an application with a unique name that other services use to refer to the functionality being called. Service instances are pods/VMs/containers that implement the service.

*Service versions* - In a continuous deployment scenario, for a given service, there can be multiple sets of instances running potentially different variants of the application binary. These variants are not necessarily different API versions. They could be iterative changes to the same service, deployed in different environments (prod, staging, dev, etc.). Common scenarios where this occurs include A/B testing, canary rollouts, etc. The choice of a particular version can be decided based on various criterion (headers, url, etc.) and/or by weights assigned to each version. Each service has a default version consisting of all its instances.

*Source* - downstream client (browser or another service) calling the Envoy proxy/sidecar (typically to reach another service).

*Destination* - The remote upstream service to which the Envoy proxy/sidecar is talking to, on behalf of the source service. There can be one or more service versions for a given service (see the discussion on versions above). Envoy would choose the version based on various routing rules.

*Access model* - Applications address only the destination service without knowledge of individual service versions. The actual choice of the version is determined by Envoy, enabling the application code to decouple itself from the evolution of dependent services.

## CircuitBreaker

Circuit breaker configuration for Envoy. The circuit breaker implementation is fine-grained in that it tracks the success/failure rates of individual hosts in the load balancing pool. Hosts that continually return errors for API calls are ejected from the pool for a pre-defined period of time. See Envoy’s [circuit breaker](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/circuit_breaking) and [outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/outlier) for more details.

| Field      | Type                                                | Description                                                  |
| ---------- | --------------------------------------------------- | ------------------------------------------------------------ |
| `simpleCb` | `CircuitBreaker.SimpleCircuitBreakerPolicy (oneof)` |                                                              |
| `custom`   | `google.protobuf.Any (oneof)`                       | (– For proxies that support custom circuit breaker policies. –) |

## CircuitBreaker.SimpleCircuitBreakerPolicy

A simple circuit breaker can be set based on a number of criteria such as connection and request limits. For example, the following destination policy sets a limit of 100 connections to “reviews” service version “v1” backends.

```yaml
metadata:
  name: reviews-cb-policy
  namespace: default
spec:
  destination:
    name: reviews
    labels:
      version: v1
  circuitBreaker:
    simpleCb:
      maxConnections: 100
```

The following destination policy sets a limit of 100 connections and 1000 concurrent requests, with no more than 10 req/connection to “reviews” service version “v1” backends. In addition, it configures hosts to be scanned every 5 mins, such that any host that fails 7 consecutive times with 5XX error code will be ejected for 15 minutes.

```yaml
metadata:
  name: reviews-cb-policy
  namespace: default
spec:
  destination:
    name: reviews
    labels:
      version: v1
  circuitBreaker:
    simpleCb:
      maxConnections: 100
      httpMaxRequests: 1000
      httpMaxRequestsPerConnection: 10
      httpConsecutiveErrors: 7
      sleepWindow: 15m
      httpDetectionInterval: 5m
```

| Field                          | Type                       | Description                                                  |
| ------------------------------ | -------------------------- | ------------------------------------------------------------ |
| `maxConnections`               | `int32`                    | Maximum number of connections to a backend.                  |
| `httpMaxPendingRequests`       | `int32`                    | Maximum number of pending requests to a backend. Default 1024 |
| `httpMaxRequests`              | `int32`                    | Maximum number of requests to a backend. Default 1024        |
| `sleepWindow`                  | `google.protobuf.Duration` | Minimum time the circuit will be open. format: 1h/1m/1s/1ms. MUST BE >=1ms. Default is 30s. |
| `httpConsecutiveErrors`        | `int32`                    | Number of 5XX errors before circuit is opened. Defaults to 5. |
| `httpDetectionInterval`        | `google.protobuf.Duration` | Time interval between ejection sweep analysis. format: 1h/1m/1s/1ms. MUST BE >=1ms. Default is 10s. |
| `httpMaxRequestsPerConnection` | `int32`                    | Maximum number of requests per connection to a backend. Setting this parameter to 1 disables keep alive. |
| `httpMaxEjectionPercent`       | `int32`                    | Maximum % of hosts in the load balancing pool for the destination service that can be ejected by the circuit breaker. Defaults to 10%. |
| `httpMaxRetries`               | `int32`                    | Maximum number of retries that can be outstanding to all hosts in a cluster at a given time. Defaults to 3. |

## CorsPolicy

Describes the Cross-Origin Resource Sharing (CORS) policy, for a given service. Refer to https://developer.mozilla.org/en-US/docs/Web/HTTP/Access*control*CORS for further details about cross origin resource sharing. For example, the following rule restricts cross origin requests to those originating from example.com domain using HTTP POST/GET, and sets the Access-Control-Allow-Credentials header to false. In addition, it only exposes X-Foo-bar header and sets an expiry period of 1 day.

```yaml
metadata:
  name: my-rule
  namespace: default
spec:
  destination:
    name: ratings
  route:
  - labels:
      version: v1
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

## DestinationPolicy

DestinationPolicy defines client/caller-side policies that determine how to handle traffic bound to a particular destination service. The policy specifies configuration for load balancing and circuit breakers. For example, a simple load balancing policy for the ratings service would look as follows:

```yaml
metadata:
  name: ratings-lb-policy
  namespace: default # optional (default is "default")
spec:
  destination:
    name: ratings
  loadBalancing:
    name: ROUND_ROBIN
```

The FQDN of the destination service is composed from the destination name and meta namespace fields, along with a platform-specific domain suffix (e.g. on Kubernetes, “reviews” + “default” + “svc.cluster.local” -> “reviews.default.svc.cluster.local”).

A destination policy can be restricted to a particular version of a service or applied to all versions. It can also be restricted to calls from a particular source. For example, the following load balancing policy applies to version v1 of the ratings service running in the prod environment but only when called from version v2 of the reviews service:

```yaml
metadata:
  name: ratings-lb-policy
  namespace: default
spec:
  source:
    name: reviews
    labels:
      version: v2
  destination:
    name: ratings
    labels:
      env: prod
      version: v1
  loadBalancing:
    name: ROUND_ROBIN
```

*Note:* Destination policies will be applied only if the corresponding tagged instances are explicitly routed to. In other words, for every destination policy defined, at least one route rule must refer to the service version indicated in the destination policy.

| Field            | Type                  | Description                                                  |
| ---------------- | --------------------- | ------------------------------------------------------------ |
| `destination`    | `IstioService`        | Optional: Destination uniquely identifies the destination service associated with this policy. |
| `source`         | `IstioService`        | Optional: Source uniquely identifies the source service associated with this policy. |
| `loadBalancing`  | `LoadBalancing`       | Load balancing policy.                                       |
| `circuitBreaker` | `CircuitBreaker`      | Circuit breaker policy.                                      |
| `custom`         | `google.protobuf.Any` | (– Other custom policy implementations –)                    |

## DestinationWeight

Each routing rule is associated with one or more service versions (see glossary in beginning of document). Weights associated with the version determine the proportion of traffic it receives. For example, the following rule will route 25% of traffic for the “reviews” service to instances with the “v2” tag and the remaining traffic (i.e., 75%) to “v1”.

```yaml
metadata:
  name: my-rule
  namespace: default
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v2
    weight: 25
  - labels:
      version: v1
    weight: 75
```

| Field         | Type                  | Description                                                  |
| ------------- | --------------------- | ------------------------------------------------------------ |
| `destination` | `IstioService`        | Sometimes required. Optional destination uniquely identifies the destination service. If not specified, the value is inherited from the parent route rule. |
| `labels`      | `map<string, string>` | Sometimes required. Service version identifier for the destination service. (– N.B. The map is used instead of pstruct due to lack of serialization support in golang protobuf library (see https://github.com/golang/protobuf/pull/208) –) |
| `weight`      | `int32`               | REQUIRED. The proportion of traffic to be forwarded to the service version. (0-100). Sum of weights across destinations SHOULD BE == 100. If there is only destination in a rule, the weight value is assumed to be 100. When using multiple weights, either destination or labels must be specified. |

## EgressRule

Egress rules describe the properties of a service outside Istio. When transparent proxying is used, egress rules signify a white listed set of external services that microserves in the mesh are allowed to access. A subset of routing rules and all destination policies can be applied on the service targeted by an egress rule. TCP services and HTTP-based services can be expressed by an egress rule. The destination of an egress rule for HTTP-based services must be an IP or a domain name, optionally with a wildcard prefix (e.g., *.foo.com). For TCP based services, the destination of an egress rule must be an IP or a block of IPs in CIDR notation.

If TLS origination from the sidecar is desired, the protocol associated with the service port must be marked as HTTPS, and the service is expected to be accessed over HTTP (e.g., http://gmail.com:443). The sidecar will automatically upgrade the connection to TLS when initiating a connection with the external service.

For example, the following egress rule describes the set of services hosted under the *.foo.com domain

```yaml
kind: EgressRule
metadata:
  name: foo-egress-rule
spec:
  destination:
    service: *.foo.com
  ports:
    - port: 80
      protocol: http
    - port: 443
      protocol: https
```

The following egress rule describes the set of services accessed by a block of IPs

```yaml
kind: EgressRule
metadata:
  name: bar-egress-rule
spec:
  destination:
    service: 92.198.174.192/27
  ports:
    - port: 111
      protocol: tcp
```

| Field            | Type                | Description                                                  |
| ---------------- | ------------------- | ------------------------------------------------------------ |
| `destination`    | `IstioService`      | REQUIRED: A domain name, optionally with a wildcard prefix, or an IP, or a block of IPs associated with the external service. ONLY the “service” field of “destination” will be taken into consideration. Name, namespace, domain and labels are ignored. Routing rules and destination policies that refer to these external services must have identical specification for the destination as the corresponding egress rule.The “service” field of “destination” for HTTP-based services must be an IP or a domain name, optionally with a wildcard prefix. Wildcard domain specifications must conform to format allowed by Envoy’s Virtual Host specification, such as “*.foo.com” or “*-bar.foo.com”. The character ‘*’ in a domain specification indicates a non-empty string. Hence, a wildcard domain of form “*-bar.foo.com” will match “baz-bar.foo.com” but not “-bar.foo.com”.The “service” field of “destination” for TCP services must be an IP or a block of IPs in CIDR notation. |
| `ports`          | `EgressRule.Port[]` | REQUIRED: list of ports on which the external service is available. |
| `useEgressProxy` | `bool`              | Forward all the external traffic through a dedicated egress proxy. It is used in some scenarios where there is a requirement that all the external traffic goes through special dedicated nodes/pods. These dedicated egress nodes could then be more closely monitored for security vulnerabilities.The default is false, i.e. the sidecar forwards external traffic directly to the external service. |

## EgressRule.Port

Port describes the properties of a specific TCP port of an external service.

| Field      | Type     | Description                                                  |
| ---------- | -------- | ------------------------------------------------------------ |
| `port`     | `int32`  | A valid non-negative integer port number.                    |
| `protocol` | `string` | The protocol to communicate with the external services. MUST BE one of HTTP\|HTTPS\|GRPC\|HTTP2\|TCP\|MONGO. |

## HTTPFaultInjection

HTTPFaultInjection can be used to specify one or more faults to inject while forwarding http requests to the destination specified in the route rule. Fault specification is part of a route rule. Faults include aborting the Http request from downstream service, and/or delaying proxying of requests. A fault rule MUST HAVE delay or abort or both.

*Note:* Delay and abort faults are independent of one another, even if both are specified simultaneously.

| Field   | Type                       | Description                                                  |
| ------- | -------------------------- | ------------------------------------------------------------ |
| `delay` | `HTTPFaultInjection.Delay` | Delay requests before forwarding, emulating various failures such as network issues, overloaded upstream service, etc. |
| `abort` | `HTTPFaultInjection.Abort` | Abort Http request attempts and return error codes back to downstream service, giving the impression that the upstream service is faulty. |

## HTTPFaultInjection.Abort

Abort specification is used to prematurely abort a request with a pre-specified error code. The following example will return an HTTP 400 error code for 10% of the requests to the “ratings” service “v1”.

```yaml
metadata:
  name: my-rule
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v1
  httpFault:
    abort:
      percent: 10
      httpStatus: 400
```

The *httpStatus* field is used to indicate the HTTP status code to return to the caller. The optional *percent* field, a value between 0 and 100, is used to only abort a certain percentage of requests. If not specified, all requests are aborted.

| Field                | Type             | Description                                                  |
| -------------------- | ---------------- | ------------------------------------------------------------ |
| `percent`            | `float`          | percentage of requests to be aborted with the error code provided (0-100). |
| `grpcStatus`         | `string (oneof)` |                                                              |
| `http2Error`         | `string (oneof)` |                                                              |
| `httpStatus`         | `int32 (oneof)`  | REQUIRED. HTTP status code to use to abort the Http request. |
| `overrideHeaderName` | `string`         | (– Specify abort code as part of Http request. TODO: The semantics and syntax of the headers is undefined. –) |

## HTTPFaultInjection.Delay

Delay specification is used to inject latency into the request forwarding path. The following example will introduce a 5 second delay in 10% of the requests to the “v1” version of the “reviews” service.

```yaml
metadata:
  name: my-rule
spec:
  destination:
    name: reviews
  route:
  - labels:
      version: v1
  httpFault:
    delay:
      percent: 10
      fixedDelay: 5s
```

The *fixedDelay* field is used to indicate the amount of delay in seconds. An optional *percent* field, a value between 0 and 100, can be used to only delay a certain percentage of requests. If left unspecified, all request will be delayed.

| Field                | Type                               | Description                                                  |
| -------------------- | ---------------------------------- | ------------------------------------------------------------ |
| `percent`            | `float`                            | percentage of requests on which the delay will be injected (0-100) |
| `fixedDelay`         | `google.protobuf.Duration (oneof)` | REQUIRED. Add a fixed delay before forwarding the request. Format: 1h/1m/1s/1ms. MUST be >=1ms. |
| `exponentialDelay`   | `google.protobuf.Duration (oneof)` | (– Add a delay (based on an exponential function) before forwarding the request. mean delay needed to derive the exponential delay values –) |
| `overrideHeaderName` | `string`                           | (– Specify delay duration as part of Http request. TODO: The semantics and syntax of the headers is undefined. –) |

## HTTPRedirect

HTTPRedirect can be used to send a 302 redirect response to the caller, where the Authority/Host and the URI in the response can be swapped with the specified values. For example, the following route rule redirects requests for /v1/getProductRatings API on the ratings service to /v1/bookRatings provided by the bookratings service.

```yaml
metadata:
  name: my-rule
  namespace: default
spec:
  destination:
    name: ratings
  match:
    request:
      headers:
        uri: /v1/getProductRatings
  redirect:
    uri: /v1/bookRatings
    authority: bookratings.default.svc.cluster.local
```

| Field       | Type     | Description                                                  |
| ----------- | -------- | ------------------------------------------------------------ |
| `uri`       | `string` | On a redirect, overwrite the Path portion of the URL with this value. Note that the entire path will be replaced, irrespective of the request URI being matched as an exact path or prefix. |
| `authority` | `string` | On a redirect, overwrite the Authority/Host portion of the URL with this value |

## HTTPRetry

Describes the retry policy to use when a HTTP request fails. For example, the following rule sets the maximum number of retries to 3 when calling ratings:v1 service, with a 2s timeout per retry attempt.

```yaml
metadata:
  name: my-rule
  namespace: default
spec:
  destination:
    name: ratings
  route:
  - labels:
      version: v1
  httpReqRetries:
    simpleRetry:
      attempts: 3
      perTryTimeout: 2s
```

| Field         | Type                                  | Description                                          |
| ------------- | ------------------------------------- | ---------------------------------------------------- |
| `simpleRetry` | `HTTPRetry.SimpleRetryPolicy (oneof)` |                                                      |
| `custom`      | `google.protobuf.Any (oneof)`         | (– For proxies that support custom retry policies –) |

## HTTPRetry.SimpleRetryPolicy

| Field                | Type                       | Description                                                  |
| -------------------- | -------------------------- | ------------------------------------------------------------ |
| `attempts`           | `int32`                    | REQUIRED. Number of retries for a given request. The interval between retries will be determined automatically (25ms+). Actual number of retries attempted depends on the httpReqTimeout. |
| `perTryTimeout`      | `google.protobuf.Duration` | Timeout per retry attempt for a given request. format: 1h/1m/1s/1ms. MUST BE >=1ms. |
| `overrideHeaderName` | `string`                   | (– Downstream Service could specify retry attempts via Http header to Envoy, if Envoy supports such a feature. –) |

## HTTPRewrite

HTTPRewrite can be used to rewrite specific parts of a HTTP request before forwarding the request to the destination. Rewrite primitive can be used only with the DestinationWeights. The following example demonstrates how to rewrite the URL prefix for api call (/ratings) to ratings service before making the actual API call.

```yaml
metadata:
  name: my-rule
  namespace: default
spec:
  destination:
    name: ratings
  match:
    request:
      headers:
        uri:
          prefix: /ratings
  rewrite:
    uri: /v1/bookRatings
  route:
  - labels:
      version: v1
```

| Field       | Type     | Description                                                  |
| ----------- | -------- | ------------------------------------------------------------ |
| `uri`       | `string` | rewrite the Path (or the prefix) portion of the URI with this value. If the original URI was matched based on prefix, the value provided in this field will replace the corresponding matched prefix. |
| `authority` | `string` | rewrite the Authority/Host header with this value.           |

## HTTPTimeout

Describes HTTP request timeout. For example, the following rule sets a 10 second timeout for calls to the ratings:v1 service

```yaml
metadata:
  name: my-rule
  namespace: default
spec:
  destination:
    name: ratings
  route:
  - labels:
      version: v1
  httpReqTimeout:
    simpleTimeout:
      timeout: 10s
```

| Field           | Type                                      | Description                                            |
| --------------- | ----------------------------------------- | ------------------------------------------------------ |
| `simpleTimeout` | `HTTPTimeout.SimpleTimeoutPolicy (oneof)` |                                                        |
| `custom`        | `google.protobuf.Any (oneof)`             | (– For proxies that support custom timeout policies –) |

## HTTPTimeout.SimpleTimeoutPolicy

| Field                | Type                       | Description                                                  |
| -------------------- | -------------------------- | ------------------------------------------------------------ |
| `timeout`            | `google.protobuf.Duration` | REQUIRED. Timeout for a HTTP request. Includes retries as well. Default 15s. format: 1h/1m/1s/1ms. MUST BE >=1ms. It is possible to control timeout per request by supplying the timeout value via x-envoy-upstream-rq-timeout-ms HTTP header. |
| `overrideHeaderName` | `string`                   | (– Downstream service could specify timeout via Http header to Envoy, if Envoy supports such a feature. –) |

## IngressRule

Ingress rules are routing rules applied to the ingress proxy pool. The ingress proxes serve as the receiving edge proxy for the entire mesh, but can also be addressed from inside the mesh. Each ingress rule defines a destination service and port. Rules that do not resolve to a service or a port in the mesh should be ignored.

The routing rules for the destination service are applied at the ingress proxy. That means the routing rule match conditions are composed and its actions are enforced. The traffic splitting for the destination service is also effective.

WARNING: This API is experimental and under active development

| Field                 | Type             | Description                                                  |
| --------------------- | ---------------- | ------------------------------------------------------------ |
| `port`                | `int32`          | REQUIRED: Port on which the ingress proxy listens and applies the rule. |
| `tlsSecret`           | `string`         | Optional TLS secret path to apply server-side TLS context on the port. It is up to the underlying secret store to interpret the path to the secret. |
| `precedence`          | `int32`          | RECOMMENDED. Precedence is used to disambiguate the order of application of rules. A higher number takes priority. If not specified, the value is assumed to be 0. The order of application for rules with the same precedence is unspecified. |
| `match`               | `MatchCondition` | Match conditions to be satisfied for the ingress rule to be activated. |
| `destination`         | `IstioService`   | REQUIRED: Destination uniquely identifies the destination service.*Note:* The ingress rule destination specification represents all version of the service and therefore the IstioService’s labels field MUST be empty. |
| `destinationPort`     | `int32 (oneof)`  | Identifies the destination service port by value             |
| `destinationPortName` | `string (oneof)` | Identifies the destination service port by name              |

## IstioService

IstioService identifies a service and optionally service version. The FQDN of the service is composed from the name, namespace, and implementation-specific domain suffix (e.g. on Kubernetes, “reviews” + “default” + “svc.cluster.local” -> “reviews.default.svc.cluster.local”).

| Field       | Type                  | Description                                                  |
| ----------- | --------------------- | ------------------------------------------------------------ |
| `name`      | `string`              | The short name of the service such as “foo”.                 |
| `namespace` | `string`              | Optional namespace of the service. Defaults to value of metadata namespace field. |
| `domain`    | `string`              | Domain suffix used to construct the service FQDN in implementations that support such specification. |
| `service`   | `string`              | The service FQDN.                                            |
| `labels`    | `map<string, string>` | Optional one or more labels that uniquely identify the service version.*Note:* When used for a RouteRule destination, labels MUST be empty. |

## L4FaultInjection

(– Faults can be injected into the connections from downstream by the Envoy, for testing the failure recovery capabilities of downstream services. Faults include aborting the connection from downstream service, delaying proxying of connection to the destination service, and throttling the bandwidth of the connection (either end). Bandwidth throttling for failure testing should not be confused with the rate limiting policy enforcement provided by the Mixer component. L4 fault injection is not supported at the moment. –)

| Field       | Type                         | Description                                                  |
| ----------- | ---------------------------- | ------------------------------------------------------------ |
| `throttle`  | `L4FaultInjection.Throttle`  | Unlike Http services, we have very little context for raw Tcp\|Udp connections. We could throttle bandwidth of the connections (slow down the connection) and/or abruptly reset (terminate) the Tcp connection after it has been established. We first throttle (if set) and then terminate the connection. |
| `terminate` | `L4FaultInjection.Terminate` |                                                              |

## L4FaultInjection.Terminate

Abruptly reset (terminate) the Tcp connection after it has been established, emulating remote server crash or link failure.

| Field                  | Type                       | Description                                                  |
| ---------------------- | -------------------------- | ------------------------------------------------------------ |
| `percent`              | `float`                    | percentage of established Tcp connections to be terminated/reset |
| `terminateAfterPeriod` | `google.protobuf.Duration` | TODO: see if it makes sense to create a generic Duration type to express time interval related configs. |

## L4FaultInjection.Throttle

Bandwidth throttling for Tcp and Udp connections

| Field                 | Type                               | Description                                                  |
| --------------------- | ---------------------------------- | ------------------------------------------------------------ |
| `percent`             | `float`                            | percentage of connections to throttle.                       |
| `downstreamLimitBps`  | `int64`                            | bandwidth limit in “bits” per second between downstream and Envoy |
| `upstreamLimitBps`    | `int64`                            | bandwidth limits in “bits” per second between Envoy and upstream |
| `throttleAfterPeriod` | `google.protobuf.Duration (oneof)` | Wait a while after the connection is established, before starting bandwidth throttling. This would allow us to inject fault after the application protocol (e.g., MySQL) has had time to establish sessions/whatever handshake necessary. |
| `throttleAfterBytes`  | `double (oneof)`                   | Alternatively, we could wait for a certain number of bytes to be transferred to upstream before throttling the bandwidth. |
| `throttleForPeriod`   | `google.protobuf.Duration`         | Stop throttling after the given duration. If not set, the connection will be throttled for its lifetime. |

## L4MatchAttributes

(– L4 connection match attributes. Note that L4 connection matching support is incomplete. –)

| Field               | Type       | Description                                                  |
| ------------------- | ---------- | ------------------------------------------------------------ |
| `sourceSubnet`      | `string[]` | IPv4 or IPv6 ip address with optional subnet. E.g., a.b.c.d/xx form or just a.b.c.d |
| `destinationSubnet` | `string[]` | IPv4 or IPv6 ip address of destination with optional subnet. E.g., a.b.c.d/xx form or just a.b.c.d. This is only valid when the destination service has several IPs and the application explicitly specifies a particular IP. |

## LoadBalancing

Load balancing policy to use when forwarding traffic. These policies directly correlate to [load balancer types](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing) supported by Envoy. Example,

```yaml
metadata:
  name: reviews-lb-policy
  namespace: default
spec:
  destination:
    name: reviews
  loadBalancing:
    name: RANDOM
```

| Field    | Type                                   | Description                                                  |
| -------- | -------------------------------------- | ------------------------------------------------------------ |
| `name`   | `LoadBalancing.SimpleLBPolicy (oneof)` | Load balancing policy name (as defined in SimpleLBPolicy below) |
| `custom` | `google.protobuf.Any (oneof)`          | (– Custom LB policy implementations –)                       |

## LoadBalancing.SimpleLBPolicy

Load balancing algorithms supported by Envoy.

| Name          | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| `ROUND_ROBIN` | Simple round robin policy.                                   |
| `LEAST_CONN`  | The least request load balancer uses an O(1) algorithm which selects two random healthy hosts and picks the host which has fewer active requests. |
| `RANDOM`      | The random load balancer selects a random healthy host. The random load balancer generally performs better than round robin if no health checking policy is configured. |

## MatchCondition

Match condition specifies a set of criterion to be met in order for the route rule to be applied to the connection or HTTP request. The condition provides distinct set of conditions for each protocol with the intention that conditions apply only to the service ports that match the protocol. For example, the following route rule restricts the rule to match only requests originating from “reviews:v2”, accessing ratings service where the URL path starts with /ratings/v2/ and the request contains a “cookie” with value “user=jason”,

```yaml
metadata:
  name: my-rule
  namespace: default
spec:
  destination:
    name: ratings
  match:
    source:
      name: reviews
      labels:
        version: v2
    request:
      headers:
        cookie:
          regex: "^(.*?;)?(user=jason)(;.*)?"
        uri:
          prefix: "/ratings/v2/"
```

MatchCondition CANNOT be empty. At least one source or request header must be specified.

| Field     | Type                | Description                                                  |
| --------- | ------------------- | ------------------------------------------------------------ |
| `source`  | `IstioService`      | Identifies the service initiating a connection or a request. |
| `tcp`     | `L4MatchAttributes` | (– Set of layer 4 match conditions based on the IP ranges –) |
| `udp`     | `L4MatchAttributes` | (– Set of layer 4 match conditions based on the IP ranges –) |
| `request` | `MatchRequest`      | Attributes of an HTTP request to match.                      |

## MatchRequest

MatchRequest specifies the attributes of an HTTP request to be used for matching a request.

| Field     | Type                       | Description                                                  |
| --------- | -------------------------- | ------------------------------------------------------------ |
| `headers` | `map<string, StringMatch>` | Set of HTTP match conditions based on HTTP/1.1, HTTP/2, GRPC request metadata, such as *uri*, *scheme*, *authority*. The header keys must be lowercase and use hyphen as the separator, e.g. *x-request-id*.Header values are case-sensitive and formatted as follows:*exact: “value”* or just *“value”* for exact string match*prefix: “value”* for prefix-based match*regex: “value”* for ECMAscript style regex-based match*Note 1:* The keys *uri*, *scheme*, *method*, and *authority* correspond to URI, protocol scheme (e.g., HTTP, HTTPS), HTTP method (e.g., GET, POST), and the HTTP Host header respectively.*Note 2:* *uri* can be used to perform URL matches. For all HTTP headers including *uri*, exact, prefix and ECMA style regular expression matches are supported. |

## RouteRule

Route rule provides a custom routing policy based on the source and destination service versions and connection/request metadata. The rule must provide a set of conditions for each protocol (TCP, UDP, HTTP) that the destination service exposes on its ports.

The rule applies only to the ports on the destination service for which it provides protocol-specific match condition, e.g. if the rule does not specify TCP condition, the rule does not apply to TCP traffic towards the destination service.

For example, a simple rule to send 100% of incoming traffic for a “reviews” service to version “v1” can be specified as follows:

```yaml
metadata:
  name: my-rule
  namespace: default # optional (default is "default")
spec:
  destination:
    name: reviews
    namespace: my-namespace # optional (default is metadata namespace field)
  route:
  - labels:
      version: v1
    weight: 100
```

| Field              | Type                  | Description                                                  |
| ------------------ | --------------------- | ------------------------------------------------------------ |
| `destination`      | `IstioService`        | REQUIRED: Destination uniquely identifies the destination associated with this routing rule. This field is applicable for hostname-based resolution for HTTP traffic as well as IP-based resolution for TCP/UDP traffic.*Note:* The route rule destination specification represents all version of the service and therefore the IstioService’s labels field MUST be empty. |
| `precedence`       | `int32`               | RECOMMENDED. Precedence is used to disambiguate the order of application of rules for the same destination service. A higher number takes priority. If not specified, the value is assumed to be 0. The order of application for rules with the same precedence is unspecified. |
| `match`            | `MatchCondition`      | Match condtions to be satisfied for the route rule to be activated. If match is omitted, the route rule applies only to HTTP traffic. |
| `route`            | `DestinationWeight[]` | REQUIRED (route\|redirect). A routing rule can either redirect traffic or forward traffic. The forwarding target can be one of several versions of a service (see glossary in beginning of document). Weights associated with the service version determine the proportion of traffic it receives. |
| `redirect`         | `HTTPRedirect`        | REQUIRED (route\|redirect). A routing rule can either redirect traffic or forward traffic. The redirect primitive can be used to send a HTTP 302 redirect to a different URI or Authority. |
| `rewrite`          | `HTTPRewrite`         | Rewrite HTTP URIs and Authority headers. Rewrite cannot be used with Redirect primitive. Rewrite will be performed before forwarding. |
| `websocketUpgrade` | `bool`                | Indicates that a HTTP/1.1 client connection to this particular route should be allowed (and expected) to upgrade to a WebSocket connection. The default is false. Envoy expects the first request to this route to contain the WebSocket upgrade headers. Otherwise, the request will be rejected. |
| `httpReqTimeout`   | `HTTPTimeout`         | Timeout policy for HTTP requests.                            |
| `httpReqRetries`   | `HTTPRetry`           | Retry policy for HTTP requests.                              |
| `httpFault`        | `HTTPFaultInjection`  | Fault injection policy to apply on HTTP traffic              |
| `l4Fault`          | `L4FaultInjection`    | (– L4 fault injection policy applies to Tcp/Udp (not HTTP) traffic –) |
| `mirror`           | `IstioService`        | Mirror HTTP traffic to a another destination in addition to forwarding the requests to the intended destination. Mirrored traffic is on best effort basis where Envoy will not wait for the mirrored cluster to respond before returning the response from the original destination. Statistics will be generated for the mirrored destination. |
| `corsPolicy`       | `CorsPolicy`          | Cross-Origin Resource Sharing policy (CORS). Refer to https://developer.mozilla.org/en-US/docs/Web/HTTP/Access*control*CORS for further details about cross origin resource sharing. |
| `appendHeaders`    | `map<string, string>` | Additional HTTP headers to add before forwarding a request to the destnation service. |

## StringMatch

Describes how to match a given string in HTTP headers. Match is case-sensitive.

| Field    | Type             | Description                        |
| -------- | ---------------- | ---------------------------------- |
| `exact`  | `string (oneof)` | exact string match                 |
| `prefix` | `string (oneof)` | prefix-based match                 |
| `regex`  | `string (oneof)` | ECMAscript style regex-based match |