---
title: Protocol support in Kuma
---

At its core, {{site.mesh_product_name}} distinguishes between the following major categories of traffic: `http`, `grpc`, `kafka` and opaque `tcp` traffic.

For `http`, `grpc` and `kafka` traffic {{site.mesh_product_name}} provides deep insights down to application-level transactions, in the latter `tcp` case the observability is limited to connection-level statistics.

So, as a user of {{site.mesh_product_name}}, you're _highly encouraged_ to give it a hint whether your service supports `http` , `grpc`, `kafka` or not.

By doing this,

{% if_version lte:2.5.x %}
* you will get richer metrics with [`Traffic Metrics`](/docs/{{ page.release }}/policies/traffic-metrics) policy
* you will get richer logs with [`Traffic Log`](/docs/{{ page.release }}/policies/traffic-log) policy
* you will be able to use [`Traffic Trace`](/docs/{{ page.release }}/policies/traffic-trace) policy
{% endif_version %}
{% if_version gte: 2.6.x %}
* you will get richer metrics with [`MeshMetric`](/docs/{{ page.release }}/policies/meshmetric) policy
* you will get richer logs with [`MeshAccessLog`](/docs/{{ page.release }}/policies/meshaccesslog) policy
* you will be able to use [`MeshTrace`](/docs/{{ page.release }}/policies/meshtrace) policy
{% endif_version %}

{% tabs %}
{% tab Kubernetes %}
On `Kubernetes`, to give {{site.mesh_product_name}} a hint that your service supports `HTTP` protocol, you need to add an `appProtocol` to the `k8s` `Service` object.

E.g.,

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: kuma-example
spec:
  selector:
    app: web
  ports:
  - port: 8080
    appProtocol: http # let {{site.mesh_product_name}} know that your service supports HTTP protocol
```

{% endtab %}
{% tab protocol-support Kubernetes (deprecated) %}
On `Kubernetes`, to give {{site.mesh_product_name}} a hint that your service supports `HTTP` protocol, you need to add a `<port>.service.kuma.io/protocol` annotation to the `k8s` `Service` object.

E.g.,

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: kuma-example
  annotations:
    8080.service.kuma.io/protocol: http # let {{site.mesh_product_name}} know that your service supports HTTP protocol
spec:
  selector:
    app: web
  ports:
  - port: 8080
```

{% endtab %}
{% tab Universal %}
On `Universal`, to give {{site.mesh_product_name}} a hint that your service supports the `http` protocol, you need to add a `kuma.io/protocol` tag to the `inbound` interface of your `Dataplane`.

E.g.,

```yaml
type: Dataplane
mesh: default
name: web
networking:
  address: 192.168.0.1 
  inbound:
  - port: 80
    servicePort: 8080
    tags:
      kuma.io/service: web
      kuma.io/protocol: http # let {{site.mesh_product_name}} know that your service supports HTTP protocol
```
{% endtab %}
{% endtabs %}

## HTTP/2 support

{{site.mesh_product_name}} by default upgrades connection between Dataplanes to HTTP/2. If you want to enable HTTP/2 on connections between a dataplane and an application, use `kuma.io/protocol: http2` tag.


## TLS support

Whenever a service already initiates a TLS request to another service - and [mutual TLS](/docs/{{ page.release }}/policies/mutual-tls) is enabled - {{site.mesh_product_name}} can enforce both TLS connections end-to-end as long as the service that is generating the TLS traffic is explicitly tagged with `tcp` [protocol](/docs/{{ page.release }}/policies/protocol-support-in-kuma) (ie: `kuma.io/protocol: tcp`).

{% tip %}
Effectively `kuma-dp` will send the raw original TLS request as-is to the final destination, while in the meanwhile it will be enforcing its own TLS connection (if [mutual TLS](/docs/{{ page.release }}/policies/mutual-tls) is enabled). Hence, the traffic must be marked as being `tcp`, so `kuma-dp` won't try to parse it.
{% endtip %}

Note that in this case no advanced HTTP or GRPC statistics or logging are available. As a best practice - since {{site.mesh_product_name}} will already secure the traffic across services via the [mutual TLS](/docs/{{ page.release }}/policies/mutual-tls) policy - we suggest disabling TLS in the original services in order to get L7 metrics and capabilities.

## Websocket support

{{site.mesh_product_name}} out of the box support's `Websocket` protocol. The service exposing `Websocket` should be marked as `tcp`.

As `Websockets` use pure `TCP` connections under the hood, your service have to be recognised by {{site.mesh_product_name}} as the `TCP` one. It's also the default behavior for {{site.mesh_product_name}} to assume the service's `inbound` interfaces are the TCP ones, so you don't have to do anything, but if you want to be explicit, you can configure your services exposing `Websocket` endpoints with `appProtocol` property. I.e.:

{% tabs %}
{% tab Kubernetes %}
```yaml
apiVersion: v1
kind: Service
metadata:
  name: websocket-server
  namespace: kuma-example
spec:
  selector:
    app: websocket-server
  ports:
  - port: 8080
    appProtocol: tcp
```

{% endtab %}
{% tab Universal %}
```yaml
type: Dataplane
mesh: default
name: websocket-server
networking:
  address: 192.168.0.1 
  inbound:
  - port: 80
    servicePort: 8080
    tags:
      kuma.io/service: websocket-server
      kuma.io/protocol: tcp
```
{% endtab %}
{% endtabs %}
