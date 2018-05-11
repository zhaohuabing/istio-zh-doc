# Mixer 客户端

## APIKey

APIKey defines the explicit configuration for generating the `request.api_key` attribute from HTTP requests.

See https://swagger.io/docs/specification/authentication/api-keys for a general overview of API keys as defined by OpenAPI.

| Field    | Type             | Description                                                  |
| -------- | ---------------- | ------------------------------------------------------------ |
| `query`  | `string (oneof)` | API Key is sent as a query parameter. `query` represents the query string parameter name.For example, `query=api_key` should be used with the following request:`GET /something?api_key=abcdef12345 `Copy |
| `header` | `string (oneof)` | API key is sent in a request header. `header` represents the header name.For example, `header=X-API-KEY` should be used with the following request:`GET /something HTTP/1.1 X-API-Key: abcdef12345 `Copy |
| `cookie` | `string (oneof)` | API key is sent in a [cookie](https://swagger.io/docs/specification/authentication/cookie-authentication),For example, `cookie=X-API-KEY` should be used for the following request:`GET /something HTTP/1.1 Cookie: X-API-KEY=abcdef12345 `Copy |

## AttributeMatch

Specifies a match clause to match Istio attributes

| Field    | Type                       | Description                                                  |
| -------- | -------------------------- | ------------------------------------------------------------ |
| `clause` | `map<string, StringMatch>` | Map of attribute names to StringMatch type. Each map element specifies one condition to match.Example:clause: source.uid: exact: SOURCE*UID request.http*method: exact: POST |

## EndUserAuthenticationPolicySpec

Determines how to apply auth policies for individual requests.

| Field  | Type    | Description                                                  |
| ------ | ------- | ------------------------------------------------------------ |
| `jwts` | `JWT[]` | List of JWT rules to valide.If the request includes a JWT it must match one of the JWT listed here matched by the issuer. If validation is successfull the follow attributes are included in requests to the mixer:`request.auth.principal - The string of the issuer (`iss`) and subject (`sub`) claims within a JWT concatenated with “/” with a percent-encoded subject value  request.auth.audiences - This should reflect the audience (`aud`) claim within matched JWT.  request.auth.presenter - The authorized presenter of the credential. This value should reflect the optional Authorized Presenter (`azp`) claim within a JWT `CopyIf no match is found the request is rejected with HTTP status code 401.JWT validation is skipped if the user’s traffic request does not include a JWT. |

## EndUserAuthenticationPolicySpecBinding

EndUserAuthenticationPolicySpecBinding defines the binding between EndUserAuthenticationPolicySpecs and one or more IstioService.

| Field      | Type                                         | Description                                                  |
| ---------- | -------------------------------------------- | ------------------------------------------------------------ |
| `services` | `IstioService[]`                             | REQUIRED. One or more services to map the listed EndUserAuthenticationPolicySpecs onto. |
| `policies` | `EndUserAuthenticationPolicySpecReference[]` | REQUIRED. One or more EndUserAuthenticationPolicySpecReference that should be mapped to the specified service(s). |

## EndUserAuthenticationPolicySpecReference

EndUserAuthenticationPolicySpecReference identifies a EndUserAuthenticationPolicySpec that is bound to a set of services.

| Field       | Type     | Description                                                  |
| ----------- | -------- | ------------------------------------------------------------ |
| `name`      | `string` | REQUIRED. The short name of the EndUserAuthenticationPolicySpec. This is the resource name defined by the metadata name field. |
| `namespace` | `string` | Optional namespace of the EndUserAuthenticationPolicySpec. Defaults to the value of the metadata namespace field. |

## HTTPAPISpec

HTTPAPISpec defines the canonical configuration for generating API-related attributes from HTTP requests based on the method and uri templated path matches. It is sufficient for defining the API surface of a service for the purposes of API attribute generation. It is not intended to represent auth, quota, documentation, or other information commonly found in other API specifications, e.g. OpenAPI.

Existing standards that define operations (or methods) in terms of HTTP methods and paths can be normalized to this format for use in Istio. For example, a simple petstore API described by OpenAPIv2 [here](https://github.com/googleapis/gnostic/blob/master/examples/v2.0/yaml/petstore-simple.yaml) can be represented with the following HTTPAPISpec.

```yaml
apiVersion: config.istio.io/v1alpha2
kind: HTTPAPISpec
metadata:
  name: petstore
  namespace: default
spec:
  attributes:
    api.service: petstore.swagger.io
    api.version: 1.0.0
  patterns:
  - attributes:
      api.operation: findPets
    httpMethod: GET
    uriTemplate: /api/pets
  - attributes:
      api.operation: addPet
    httpMethod: POST
    uriTemplate: /api/pets
  - attributes:
      api.operation: findPetById
    httpMethod: GET
    uriTemplate: /api/pets/{id}
  - attributes:
      api.operation: deletePet
    httpMethod: DELETE
    uriTemplate: /api/pets/{id}
  api_keys:
  - query: api-key
```

| Field        | Type                        | Description                                                  |
| ------------ | --------------------------- | ------------------------------------------------------------ |
| `attributes` | `istio.mixer.v1.Attributes` | List of attributes that are generated when *any* of the HTTP patterns match. This list typically includes the “api.service” and “api.version” attributes. |
| `patterns`   | `HTTPAPISpecPattern[]`      | List of HTTP patterns to match.                              |
| `apiKeys`    | `APIKey[]`                  | List of APIKey that describes how to extract an API-KEY from an HTTP request. The first API-Key match found in the list is used, i.e. ‘OR’ semantics.The following default policies are used to generate the `request.api_key`attribute if no explicit APIKey is defined.``query: key, `query: api_key`, and then `header: x-api-key` `Copy |

## HTTPAPISpecBinding

HTTPAPISpecBinding defines the binding between HTTPAPISpecs and one or more IstioService. For example, the following establishes a binding between the HTTPAPISpec `petstore` and service `foo` in namespace `bar`.

```
apiVersion: config.istio.io/v1alpha2
kind: HTTPAPISpecBinding
metadata:
  name: my-binding
  namespace: default
spec:
  services:
  - name: foo
    namespace: bar
  api_specs:
  - name: petstore
    namespace: default
```

| Field      | Type                     | Description                                                  |
| ---------- | ------------------------ | ------------------------------------------------------------ |
| `services` | `IstioService[]`         | REQUIRED. One or more services to map the listed HTTPAPISpec onto. |
| `apiSpecs` | `HTTPAPISpecReference[]` | REQUIRED. One or more HTTPAPISpec references that should be mapped to the specified service(s). The aggregate collection of match conditions defined in the HTTPAPISpecs should not overlap. |

## HTTPAPISpecPattern

HTTPAPISpecPattern defines a single pattern to match against incoming HTTP requests. The per-pattern list of attributes is generated if both the http*method and uri*template match. In addition, the top-level list of attributes in the HTTPAPISpec is also generated.

```
pattern:
- attributes
    api.operation: doFooBar
  httpMethod: GET
  uriTemplate: /foo/bar
```

| Field         | Type                        | Description                                                  |
| ------------- | --------------------------- | ------------------------------------------------------------ |
| `attributes`  | `istio.mixer.v1.Attributes` | List of attributes that are generated if the HTTP request matches the specified http*method and uri*template. This typically includes the “api.operation” attribute. |
| `httpMethod`  | `string`                    | HTTP request method to match against as defined by [rfc7231](https://tools.ietf.org/html/rfc7231#page-21). For example: GET, HEAD, POST, PUT, DELETE. |
| `uriTemplate` | `string (oneof)`            | URI template to match against as defined by [rfc6570](https://tools.ietf.org/html/rfc6570). For example, the following are valid URI templates:`/pets /pets/{id} /dictionary/{term:1}/{term} /search{?q*,lang} `Copy |
| `regex`       | `string (oneof)`            | EXPERIMENTAL:ecmascript style regex-based match as defined by [EDCA-262](http://en.cppreference.com/w/cpp/regex/ecmascript). For example,`"^/pets/(.*?)?" `Copy |

## HTTPAPISpecReference

HTTPAPISpecReference defines a reference to an HTTPAPISpec. This is typically used for establishing bindings between an HTTPAPISpec and an IstioService. For example, the following defines an HTTPAPISpecReference for service `foo` in namespace `bar`.

```
- name: foo
  namespace: bar
```

| Field       | Type     | Description                                                  |
| ----------- | -------- | ------------------------------------------------------------ |
| `name`      | `string` | REQUIRED. The short name of the HTTPAPISpec. This is the resource name defined by the metadata name field. |
| `namespace` | `string` | Optional namespace of the HTTPAPISpec. Defaults to the encompassing HTTPAPISpecBinding’s metadata namespace field. |

## HttpClientConfig

Defines the client config for HTTP.

| Field                       | Type                         | Description                                                  |
| --------------------------- | ---------------------------- | ------------------------------------------------------------ |
| `transport`                 | `TransportConfig`            | The transport config.                                        |
| `serviceConfigs`            | `map<string, ServiceConfig>` | Map of control configuration indexed by destination.service. This is used to support per-service configuration for cases where a mixerclient serves multiple services. |
| `defaultDestinationService` | `string`                     | Default destination service name if none was specified in the client request. |
| `mixerAttributes`           | `istio.mixer.v1.Attributes`  | Default attributes to send to Mixer in both Check and Report. This typically includes “destination.ip” and “destination.uid” attributes. |
| `forwardAttributes`         | `istio.mixer.v1.Attributes`  | Default attributes to forward to upstream. This typically includes the “source.ip” and “source.uid” attributes. |

## IstioService

IstioService identifies a service and optionally service version. The FQDN of the service is composed from the name, namespace, and implementation-specific domain suffix (e.g. on Kubernetes, “reviews” + “default” + “svc.cluster.local” -> “reviews.default.svc.cluster.local”).

| Field       | Type                  | Description                                                  |
| ----------- | --------------------- | ------------------------------------------------------------ |
| `name`      | `string`              | The short name of the service such as “foo”.                 |
| `namespace` | `string`              | Optional namespace of the service. Defaults to value of metadata namespace field. |
| `domain`    | `string`              | Domain suffix used to construct the service FQDN in implementations that support such specification. |
| `service`   | `string`              | The service FQDN.                                            |
| `labels`    | `map<string, string>` | Optional one or more labels that uniquely identify the service version.*Note:* When used for a RouteRule destination, labels MUST be empty. |

## JWT

JSON Web Token (JWT) token format for authentication as defined by https://tools.ietf.org/html/rfc7519. See [OAuth 2.0](https://tools.ietf.org/html/rfc6749) and [OIDC 1.0](http://openid.net/connect) for how this is used in the whole authentication flow.

Example,

```
issuer: https://example.com
audiences:
- bookstore_android.apps.googleusercontent.com
  bookstore_web.apps.googleusercontent.com
jwks_uri: https://example.com/.well-known/jwks.json
```

| Field                    | Type                       | Description                                                  |
| ------------------------ | -------------------------- | ------------------------------------------------------------ |
| `issuer`                 | `string`                   | Identifies the principal that issued the JWT. See https://tools.ietf.org/html/rfc7519#section-4.1.1 Usually a URL or an email address.Example: https://securetoken.google.com Example: 1234567-compute@developer.gserviceaccount.com |
| `audiences`              | `string[]`                 | The list of JWT [audiences](https://tools.ietf.org/html/rfc7519#section-4.1.3). that are allowed to access. A JWT containing any of these audiences will be accepted.The service name will be accepted if audiences is empty.Example:`audiences: - bookstore_android.apps.googleusercontent.com   bookstore_web.apps.googleusercontent.com `Copy |
| `jwksUri`                | `string`                   | URL of the provider’s public key set to validate signature of the JWT. See [OpenID Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderMetadata).Optional if the key set document can either (a) be retrieved from [OpenID Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) of the issuer or (b) inferred from the email domain of the issuer (e.g. a Google service account).Example: https://www.googleapis.com/oauth2/v1/certs |
| `forwardJwt`             | `bool`                     | If true, forward the entire base64 encoded JWT in the HTTP request. If false, remove the JWT from the HTTP request and do not forward to the application. |
| `publicKeyCacheDuration` | `google.protobuf.Duration` | Duration after which the cached public key should be expired. The system wide default is applied if no duration is explicitly specified. |
| `locations`              | `JWT.Location[]`           | Zero or more locations to search for JWT in an HTTP request. |
| `jwksUriEnvoyCluster`    | `string`                   | This field is specific for Envoy proxy implementation. It is the cluster name in the Envoy config for the jwks_uri. |

## JWT.Location

Defines where to extract the JWT from an HTTP request.

If no explicit location is specified the following default locations are tried in order:

```
1) The Authorization header using the Bearer schema,
   e.g. Authorization: Bearer <token>. (see
   https://tools.ietf.org/html/rfc6750#section-2.1)

2) `access_token` query parameter (see
https://tools.ietf.org/html/rfc6750#section-2.3)
```

| Field    | Type             | Description                                                  |
| -------- | ---------------- | ------------------------------------------------------------ |
| `header` | `string (oneof)` | JWT is sent in a request header. `header` represents the header name.For example, if `header=x-goog-iap-jwt-assertion`, the header format will be x-goog-iap-jwt-assertion: . |
| `query`  | `string (oneof)` | JWT is sent in a query parameter. `query` represents the query parameter name.For example, `query=jwt_token`. |

## NetworkFailPolicy

Specifies the behavior when the client is unable to connect to Mixer.

| Field    | Type                           | Description                                                  |
| -------- | ------------------------------ | ------------------------------------------------------------ |
| `policy` | `NetworkFailPolicy.FailPolicy` | Specifies the behavior when the client is unable to connect to Mixer. |

## NetworkFailPolicy.FailPolicy

Describes the policy.

| Name         | Description                                                  |
| ------------ | ------------------------------------------------------------ |
| `FAIL_OPEN`  | If network connection fails, request is allowed and delivered to the service. |
| `FAIL_CLOSE` | If network connection fails, request is rejected.            |

## Quota

Specifies a quota to use with quota name and amount.

| Field    | Type     | Description                |
| -------- | -------- | -------------------------- |
| `quota`  | `string` | The quota name to charge   |
| `charge` | `int64`  | The quota amount to charge |

## QuotaRule

Specifies a rule with list of matches and list of quotas. If any clause matched, the list of quotas will be used.

| Field    | Type               | Description                                                  |
| -------- | ------------------ | ------------------------------------------------------------ |
| `match`  | `AttributeMatch[]` | If empty, match all request. If any of match is true, it is matched. |
| `quotas` | `Quota[]`          | The list of quotas to charge.                                |

## QuotaSpec

Determines the quotas used for individual requests.

| Field   | Type          | Description            |
| ------- | ------------- | ---------------------- |
| `rules` | `QuotaRule[]` | A list of Quota rules. |

## QuotaSpecBinding

QuotaSpecBinding defines the binding between QuotaSpecs and one or more IstioService.

| Field        | Type                                    | Description                                                  |
| ------------ | --------------------------------------- | ------------------------------------------------------------ |
| `services`   | `IstioService[]`                        | REQUIRED. One or more services to map the listed QuotaSpec onto. |
| `quotaSpecs` | `QuotaSpecBinding.QuotaSpecReference[]` | REQUIRED. One or more QuotaSpec references that should be mapped to the specified service(s). The aggregate collection of match conditions defined in the QuotaSpecs should not overlap. |

## QuotaSpecBinding.QuotaSpecReference

QuotaSpecReference uniquely identifies the QuotaSpec used in the Binding.

| Field       | Type     | Description                                                  |
| ----------- | -------- | ------------------------------------------------------------ |
| `name`      | `string` | REQUIRED. The short name of the QuotaSpec. This is the resource name defined by the metadata name field. |
| `namespace` | `string` | Optional namespace of the QuotaSpec. Defaults to the value of the metadata namespace field. |

## ServiceConfig

Defines the per-service client configuration.

| Field                | Type                              | Description                                                  |
| -------------------- | --------------------------------- | ------------------------------------------------------------ |
| `disableCheckCalls`  | `bool`                            | If true, do not call Mixer Check.                            |
| `disableReportCalls` | `bool`                            | If true, do not call Mixer Report.                           |
| `mixerAttributes`    | `istio.mixer.v1.Attributes`       | Send these attributes to Mixer in both Check and Report. This typically includes the “destination.service” attribute. |
| `httpApiSpec`        | `HTTPAPISpec[]`                   | HTTP API specifications to generate API attributes.          |
| `quotaSpec`          | `QuotaSpec[]`                     | Quota specifications to generate quota requirements.         |
| `endUserAuthnSpec`   | `EndUserAuthenticationPolicySpec` | End user authentication policy.                              |
| `networkFailPolicy`  | `NetworkFailPolicy`               | Specifies the behavior when the client is unable to connect to Mixer. This is the service-level policy. It overrides [mesh-level policy](https://istio.io/docs/reference/config/istio.mixer.v1.config.client.html#TransportConfig.network_fail_policy). |

## StringMatch

Describes how to match a given string in HTTP headers. Match is case-sensitive.

| Field    | Type             | Description                        |
| -------- | ---------------- | ---------------------------------- |
| `exact`  | `string (oneof)` | exact string match                 |
| `prefix` | `string (oneof)` | prefix-based match                 |
| `regex`  | `string (oneof)` | ECMAscript style regex-based match |

## TcpClientConfig

Defines the client config for TCP.

| Field                 | Type                        | Description                                                  |
| --------------------- | --------------------------- | ------------------------------------------------------------ |
| `transport`           | `TransportConfig`           | The transport config.                                        |
| `mixerAttributes`     | `istio.mixer.v1.Attributes` | Default attributes to send to Mixer in both Check and Report. This typically includes “destination.ip” and “destination.uid” attributes. |
| `disableCheckCalls`   | `bool`                      | If set to true, disables mixer check calls.                  |
| `disableReportCalls`  | `bool`                      | If set to true, disables mixer check calls.                  |
| `connectionQuotaSpec` | `QuotaSpec`                 | Quota specifications to generate quota requirements. It applies on the new TCP connections. |
| `reportInterval`      | `google.protobuf.Duration`  | Specify report interval to send periodical reports for long TCP connections. If not specified, the interval is 10 seconds. This interval should not be less than 1 second, otherwise it will be reset to 1 second. |

## TransportConfig

Defines the transport config on how to call Mixer.

| Field                 | Type                       | Description                                                  |
| --------------------- | -------------------------- | ------------------------------------------------------------ |
| `disableCheckCache`   | `bool`                     | The flag to disable check cache.                             |
| `disableQuotaCache`   | `bool`                     | The flag to disable quota cache.                             |
| `disableReportBatch`  | `bool`                     | The flag to disable report batch.                            |
| `networkFailPolicy`   | `NetworkFailPolicy`        | Specifies the behavior when the client is unable to connect to Mixer. This is the mesh level policy. The default value for policy is FAIL_OPEN. |
| `statsUpdateInterval` | `google.protobuf.Duration` | Specify refresh interval to write mixer client statistics to Envoy share memory. If not specified, the interval is 10 seconds. |
| `checkCluster`        | `string`                   | Name of the cluster that will forward check calls to a pool of mixer servers. Defaults to “mixer_server”. By using different names for checkCluster and reportCluster, it is possible to have one set of mixer servers handle check calls, while another set of mixer servers handle report calls.NOTE: Any value other than the default “mixer_server” will require the Istio Grafana dashboards to be reconfigured to use the new name. |
| `reportCluster`       | `string`                   | Name of the cluster that will forward report calls to a pool of mixer servers. Defaults to “mixer_server”. By using different names for checkCluster and reportCluster, it is possible to have one set of mixer servers handle check calls, while another set of mixer servers handle report calls.NOTE: Any value other than the default “mixer_server” will require the Istio Grafana dashboards to be reconfigured to use the new name. |

## istio.mixer.v1.Attributes

Attributes represents a set of typed name/value pairs. Many of Mixer’s API either consume and/or return attributes.

Istio uses attributes to control the runtime behavior of services running in the service mesh. Attributes are named and typed pieces of metadata describing ingress and egress traffic and the environment this traffic occurs in. An Istio attribute carries a specific piece of information such as the error code of an API request, the latency of an API request, or the original IP address of a TCP connection. For example:

```
request.path: xyz/abc
request.size: 234
request.time: 12:34:56.789 04/17/2017
source.ip: 192.168.0.1
target.service: example
```

A given Istio deployment has a fixed vocabulary of attributes that it understands. The specific vocabulary is determined by the set of attribute producers being used in the deployment. The primary attribute producer in Istio is Envoy, although specialized Mixer adapters and services can also generate attributes.

The common baseline set of attributes available in most Istio deployments is defined [here](https://istio.io/docs/reference/config/mixer/attribute-vocabulary.html).

Attributes are strongly typed. The supported attribute types are defined by [ValueType](https://github.com/istio/api/blob/master/policy/v1beta1/value_type.proto). Each type of value is encoded into one of the so-called transport types present in this message.

Defines a map of attributes in uncompressed format. Following places may use this message: 1) Configure Istio/Proxy with static per-proxy attributes, such as source.uid. 2) Service IDL definition to extract api attributes for active requests. 3) Forward attributes from client proxy to server proxy for HTTP requests.

| Field        | Type                                                    | Description                           |
| ------------ | ------------------------------------------------------- | ------------------------------------- |
| `attributes` | `map<string, istio.mixer.v1.Attributes.AttributeValue>` | A map of attribute name to its value. |