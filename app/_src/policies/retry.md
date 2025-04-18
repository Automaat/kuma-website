---
title: Retry
---
{% if_version gte:2.6.x %}
{% warning %}
New to Kuma? Don't use this policy, check [`MeshRetry`](/docs/{{ page.release }}/policies/meshretry) instead. If you want to use the `Retry` policy, remember that it requires the [TrafficRoute](/docs/{{ page.release }}/policies/traffic-route) policy to function properly.
{% endwarning %}
{% endif_version %}

{% tip %}
Retry is an outbound policy. Dataplanes whose configuration is modified are in the `sources` matcher.
{% endtip %}

This policy enables {{site.mesh_product_name}} to know how to behave if there is a failed scenario (i.e. HTTP request) which could be retried.

## Usage

As usual, we can apply `sources` and `destinations` selectors to determine how retries will be performed across our data plane proxies.

The policy let you configure retry behaviour for `HTTP`, `GRPC` and `TCP` protocols.

### Example

{% tabs %}
{% tab Kubernetes %}
```yaml
apiVersion: kuma.io/v1alpha1
kind: Retry
mesh: default
metadata:
  name: web-to-backend-retry-policy
spec:
  sources:
  - match:
      kuma.io/service: web_default_svc_80
  destinations:
  - match:
      kuma.io/service: backend_default_svc_80
  conf:
    http:
      numRetries: 5
      perTryTimeout: 200ms
      backOff:
        baseInterval: 20ms
        maxInterval: 1s
      retriableStatusCodes:
      - 500
      - 504
      retriableMethods:
      - GET
    grpc:
      numRetries: 5
      perTryTimeout: 200ms
      backOff:
        baseInterval: 20ms
        maxInterval: 1s
      retryOn:
      - cancelled
      - deadline_exceeded
      - internal
      - resource_exhausted
      - unavailable
    tcp:
      maxConnectAttempts: 3
```
We will apply the configuration with `kubectl apply -f [..]`.
{% endtab %}

{% tab Universal %}
```yaml
type: Retry
name: web-to-backend-retry-policy
mesh: default
sources:
- match:
    kuma.io/service: web
destinations:
- match:
    kuma.io/service: backend
conf:
  http:
    numRetries: 5
    perTryTimeout: 200ms
    backOff:
      baseInterval: 20ms
      maxInterval: 1s
    retriableStatusCodes:
    - 500
    - 504
    retriableMethods:
    - GET
    - DELETE
  grpc:
    numRetries: 5
    perTryTimeout: 200ms
    backOff:
      baseInterval: 20ms
      maxInterval: 1s
    retryOn:
    - cancelled
    - deadline_exceeded
    - internal
    - resource_exhausted
    - unavailable
  tcp:
    maxConnectAttempts: 3
```

We will apply the configuration with `kumactl apply -f [..]` or via the [HTTP API](/docs/{{ page.release }}/reference/http-api).
{% endtab %}
{% endtabs %}


### HTTP

- **`numRetries`** (optional)

  Amount of attempts which will be made on failed (and retriable) requests

- **`perTryTimeout`** (optional)

  Amount of time after which retry attempt should timeout (i.e. all the values: `30000000ns`, `30ms`, `0.03s`, `0.0005m` are equivalent and can be used to express the same timeout value, equal to `30ms`)

- **`backOff`** (optional)

  Configuration of durations which will be used in exponential backoff strategy between retries

  - **`baseInterval`** (required)

    Base amount of time which should be taken between retries (i.e. `30ms`, `0.03s`, `0.0005m`)

  - **`maxInterval`** (optional)

    A maximal amount of time which will be taken between retries  (i.e. `1s`, `0.5m`)

- **`retriableStatusCodes`** (optional)

  A list of status codes which will cause the request to be retried. When this field will be provided it will overwrite the default behaviour of accepting as retriable codes: `502`, `503` and `504` and if they also should be considered as `retriable` you have to manually place them in the list

  For example to add a status code `418`:
  ```yaml
  retriableStatusCodes:
  - 418
  - 502
  - 503
  - 504
  ```

{% capture tooltip %}
{% tip %}
If both `retriableStatusCodes` is provided in addition to `retryOn` (below), but the latter doesn't contain `retriable_status_codes` as a condition, it will be automatically added.
{% endtip %}
{% endcapture %}
{{ tooltip | indent }}

- **`retryOn`** (optional)

  List of conditions which will cause a retry.

  **Acceptable values**

  - `all_5xx`
  - `gateway_error`
  - `reset`
  - `connect_failure`
  - `envoy_ratelimited`
  - `retriable_4xx`
  - `refused_stream`
  - `retriable_status_codes`
  - `retriable_headers`
  - `http3_post_connect_failure`

{% capture tooltips %}
{% tip %}
Note that if `retryOn` is not defined or if it's empty, the policy will default to the equivalent of:

```yaml
retryOn:
- gateway_error
- connect_failure
- refused_stream
```
{% endtip %}

{% warning %}
Providing `retriable_status_codes` without also providing 
`retriableStatusCodes` (above) will fail policy validation.
{% endwarning %}
{% endcapture %}
{{ tooltips | indent }}

- **`retriableMethods`** (optional)

  A list of HTTP methods in which a request's method must be contained before that request can be retried. The default behavior is that all methods are retriable.


### GRPC

You can configure your GRPC Retry policy in similar fashion as the HTTP one with the only difference of the `retryOn` property which replace the `retriableStatusCodes` from the HTTP policy

- **`retryOn`** (optional)

  List of values which will cause retry.

  **Acceptable values**

  - `cancelled`
  - `deadline_exceeded`
  - `internal`
  - `resource_exhausted`
  - `unavailable`

{% capture tooltip %}
{% tip %}
Note that if `retryOn` is not defined or if it's empty, the policy will default to all values and is equivalent to:

```yaml
retryOn:
- cancelled
- deadline_exceeded
- internal
- resource_exhausted
- unavailable
```
{% endtip %}
{% endcapture %}
{{ tooltip | indent }}

### TCP

- **`maxConnectAmount`** (required)

  A maximal amount of TCP connection attempts which will be made before giving up

{% capture tooltip %}
{% tip %}
This policy will make attempt to retry the TCP connection which fail to be established and will be applied in the scenario when both, the dataplane, and the TCP service matched as a destination will be down.
{% endtip %}
{% endcapture %}
{{ tooltip | indent }}

## Matching

`Retry` is an [Outbound Connection Policy](/docs/{{ page.release }}/policies/how-kuma-chooses-the-right-policy-to-apply/#outbound-connection-policy).
The only supported value for `destinations.match` is `kuma.io/service`.

## Builtin Gateway support

Retries can be configured on each route by matching the `Retry` connection policy to the backend destination tags.

## All options

{% json_schema Retry type=proto %}
