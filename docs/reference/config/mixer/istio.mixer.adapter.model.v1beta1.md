# Mixer 适配器模型

This package defines the service and types used by adapter code to serve requests from Mixer. This package also defines the types that are used to create Mixer templates.

## Services

### InfrastructureBackend

`InfrastructureBackend` is implemented by backends that wants to provide telemetry and policy functionality to Mixer as an out of process adapter.

`InfrastructureBackend` allows Mixer and the backends to have a session based model. In a session based model, Mixer passes the relevant configuration state to the backend only once and estabilshes a session using a session identifier. All future communications between Mixer and the backend uses the session identifier which refers to the previously passed in configuration data.

For a given `InfrastructureBackend`, Mixer calls the `Validate` function, followed by `CreateSession`. The `session_id`returned from `CreateSession` is used to make subsequent request-time Handle calls and the eventual `CloseSession`function. Mixer assumes that, given the `session_id`, the backend can retrieve the necessary context to serve the Handle calls that contains the request-time payload (template-specific instance protobuf).

```
rpc Validate(ValidateRequest) returns (ValidateResponse)
```

Validates the handler configuration along with the template-specific instances that would be routed to that handler. The `CreateSession` for a specific handler configuration is invoked only if its associated `Validate` call has returned success.

```
rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse)
```

Creates a session for a given handler configuration and the template-specific instances that would be routed to that handler. For every handler configuration, Mixer creates a separate session by invoking `CreateSession` on the backend.

`CreateSessionRequest` contains the adapter specific handler configuration and the inferred type information about the instances the handler would receive during request processing.

`CreateSession` must return a `session_id` which Mixer uses to invoke template-specific Handle functions during request processing. The `session_id` provides the Handle functions a way to retrieve the necessary configuration associated with the session. Upon Mixer configuration change, Mixer will re-invoke `CreateSession` for all handler configurations whose existing sessions are invalidated or didn’t existed.

Backend is allowed to return the same session id if given the same configuration block. This would happen when multiple instances of Mixer in a deployment all create sessions with the same configuration. Note that given individial instances of Mixer can call `CloseSession`, reusing `session_id` by the backend assumes that the backend is doing reference counting.

If the backend couldn’t create a session for a specific handler configuration and returns non S_OK status, Mixer will not make request-time Handle calls associated with that handler configuration.

```
rpc CloseSession(CloseSessionRequest) returns (CloseSessionResponse)
```

Closes the session associated with the `session_id`. Mixer closes a session when its associated handler configuration or the instance configuration changes. Backend is supposed to cleanup all the resources associated with the session_id referenced by CloseSessionRequest.

## Types

### CheckResult

Expresses the result of a precondition check.

| Field           | Type                       | Description                                                  |
| --------------- | -------------------------- | ------------------------------------------------------------ |
| `status`        | `google.rpc.Status`        | A status code of OK indicates preconditions were satisfied. Any other code indicates preconditions were not satisfied and details describe why. |
| `validDuration` | `google.protobuf.Duration` | The amount of time for which this result can be considered valid. |
| `validUseCount` | `int32`                    | The number of uses for which this result can be considered valid. |

### CloseSessionRequest

Request message for `CloseSession` method.

| Field       | Type     | Description                     |
| ----------- | -------- | ------------------------------- |
| `sessionId` | `string` | Id of the session to be closed. |

### CloseSessionResponse

Response message for `CloseSession` method.

| Field    | Type                | Description                                       |
| -------- | ------------------- | ------------------------------------------------- |
| `status` | `google.rpc.Status` | The success/failure status of close session call. |

### CreateSessionRequest

Request message for `CreateSession` method.

| Field           | Type                               | Description                                                  |
| --------------- | ---------------------------------- | ------------------------------------------------------------ |
| `adapterConfig` | `google.protobuf.Any`              | Adapter specific configuration.                              |
| `inferredTypes` | `map<string, google.protobuf.Any>` | Map of instance names to their template-specific inferred type. |

### CreateSessionResponse

Response message for `CreateSession` method.

| Field       | Type                | Description                                        |
| ----------- | ------------------- | -------------------------------------------------- |
| `sessionId` | `string`            | Id of the created session.                         |
| `status`    | `google.rpc.Status` | The success/failure status of create session call. |

### DNSName

DNSName is used inside templates for fields that are of ValueType “DNS_NAME”

### Duration

Duration is used inside templates for fields that are of ValueType “DURATION”

### EmailAddress

EmailAddress is used inside templates for fields that are of ValueType “EMAIL_ADDRESS” DO NOT USE !! Under Development

### IPAddress

IPAddress is used inside templates for fields that are of ValueType “IP_ADDRESS”

### Info

Info describes an adapter or a backend that wants to provide telemetry and policy functionality to Mixer as an out of process adapter.

| Field         | Type       | Description                                                  |
| ------------- | ---------- | ------------------------------------------------------------ |
| `name`        | `string`   | Name of the adapter. It must be an RFC 1035 compatible DNS label matching the `^[a-z]([-a-z0-9]*[a-z0-9])?$` regular expression. Name is used in Istio configuration, therefore it should be descriptive but short. example: denier Vendor adapters should use a vendor prefix. example: mycompany-denier |
| `description` | `string`   | User-friendly description of the adapter.                    |
| `templates`   | `string[]` | Base64 encoded proto descriptor of all the templates the adapter wants to serve. |
| `config`      | `string`   | Base64 encoded proto descriptor of the adapter configuration. |

### QuotaRequest

Expresses the quota allocation request.

| Field    | Type                                    | Description                       |
| -------- | --------------------------------------- | --------------------------------- |
| `quotas` | `map<string, QuotaRequest.QuotaParams>` | The individual quotas to allocate |

### QuotaRequest.QuotaParams

parameters for a quota allocation

| Field        | Type    | Description                                                  |
| ------------ | ------- | ------------------------------------------------------------ |
| `amount`     | `int64` | Amount of quota to allocate                                  |
| `bestEffort` | `bool`  | When true, supports returning less quota than what was requested. |

### QuotaResult

Expresses the result of multiple quota allocations.

| Field    | Type                              | Description                                         |
| -------- | --------------------------------- | --------------------------------------------------- |
| `quotas` | `map<string, QuotaResult.Result>` | The resulting quota, one entry per requested quota. |

### QuotaResult.Result

Expresses the result of a quota allocation.

| Field           | Type                       | Description                                                  |
| --------------- | -------------------------- | ------------------------------------------------------------ |
| `validDuration` | `google.protobuf.Duration` | The amount of time for which this result can be considered valid. |
| `grantedAmount` | `int64`                    | The amount of granted quota. When `QuotaParams.best_effort` is true, this will be >= 0. If `QuotaParams.best_effort` is false, this will be either 0 or >= `QuotaParams.amount`. |

### ReportResult

Expresses the result of a report call.

### TemplateVariety

The available varieties of templates, controlling the semantics of what an adapter does with each instance.

| Name                                   | Description                                             |
| -------------------------------------- | ------------------------------------------------------- |
| `TEMPLATE_VARIETY_CHECK`               | Makes the template applicable for Mixer’s check calls.  |
| `TEMPLATE_VARIETY_REPORT`              | Makes the template applicable for Mixer’s report calls. |
| `TEMPLATE_VARIETY_QUOTA`               | Makes the template applicable for Mixer’s quota calls.  |
| `TEMPLATE_VARIETY_ATTRIBUTE_GENERATOR` | Makes the template applicable for Mixer’s quota calls.  |

### TimeStamp

TimeStamp is used inside templates for fields that are of ValueType “TIMESTAMP”

### Uri

Uri is used inside templates for fields that are of ValueType “URI” DO NOT USE ! Under Development

### ValidateRequest

Request message for `Validate` method.

| Field           | Type                               | Description                                                  |
| --------------- | ---------------------------------- | ------------------------------------------------------------ |
| `adapterConfig` | `google.protobuf.Any`              | Adapter specific configuration.                              |
| `inferredTypes` | `map<string, google.protobuf.Any>` | Map of instance names to their template-specific inferred type. |

### ValidateResponse

Response message for `Validate` method.

| Field    | Type                | Description                                    |
| -------- | ------------------- | ---------------------------------------------- |
| `status` | `google.rpc.Status` | The success/failure status of validation call. |

### Value

Value is used inside templates for fields that have dynamic types. The actual datatype of the field depends on the datatype of the expression used in the operator configuration.

### google.rpc.Status

The `Status` type defines a logical error model that is suitable for different programming environments, including REST APIs and RPC APIs. It is used by [gRPC](https://github.com/grpc). The error model is designed to be:

- Simple to use and understand for most users
- Flexible enough to meet unexpected needs

#### Overview

The `Status` message contains three pieces of data: error code, error message, and error details. The error code should be an enum value of *google.rpc.Code*, but it may accept additional error codes if needed. The error message should be a developer-facing English message that helps developers *understand* and *resolve* the error. If a localized user-facing error message is needed, put the localized message in the error details or localize it in the client. The optional error details may contain arbitrary information about the error. There is a predefined set of error detail types in the package `google.rpc` that can be used for common error conditions.

#### Language mapping

The `Status` message is the logical representation of the error model, but it is not necessarily the actual wire format. When the `Status` message is exposed in different client libraries and different wire protocols, it can be mapped differently. For example, it will likely be mapped to some exceptions in Java, but more likely mapped to some error codes in C.

#### Other uses

The error model and the `Status` message can be used in a variety of environments, either with or without APIs, to provide a consistent developer experience across different environments.

Example uses of this error model include:

- Partial errors. If a service needs to return partial errors to the client, it may embed the `Status` in the normal response to indicate the partial errors.
- Workflow errors. A typical workflow has multiple steps. Each step may have a `Status` message for error reporting.
- Batch operations. If a client uses batch request and batch response, the `Status` message should be used directly inside batch response, one for each error sub-response.
- Asynchronous operations. If an API call embeds asynchronous operation results in its response, the status of those operations should be represented directly using the `Status` message.
- Logging. If some API errors are stored in logs, the message `Status` could be used directly after any stripping needed for security/privacy reasons.

| Field     | Type                    | Description                                                  |
| --------- | ----------------------- | ------------------------------------------------------------ |
| `code`    | `int32`                 | The status code, which should be an enum value of *google.rpc.Code*. |
| `message` | `string`                | A developer-facing error message, which should be in English. Any user-facing error message should be localized and sent in the [google.rpc.Status.details](https://istio.io/docs/reference/config/mixer/istio.mixer.adapter.model.v1beta1.html#google.rpc.Status.details) field, or localized by the client. |
| `details` | `google.protobuf.Any[]` | A list of messages that carry the error details. There is a common set of message types for APIs to use. |