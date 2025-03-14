---
title: Service Health Probes
---

{{site.mesh_product_name}} is able to track the status of the Envoy proxy and the underlying app.

Compared to `HealthCheck` policies, Health Probes have the following advantages:

- knowledge about health is propagated back to `kuma-cp` and is visible in both in {{site.mesh_product_name}} GUI and {{site.mesh_product_name}} CLI
- scalable with thousands of data plane proxies

Unlike `HealthCheck` policies, Health Probes:
- only updates when `kuma-cp` is up and running and the proxy can connect to it.
- doesn't check connectivity between data plane proxies.

Every `inbound` in the `Dataplane` model has a `health` section:

```yaml
type: Dataplane
mesh: default
name: web-01
networking:
  address: 127.0.0.1
  inbound:
    - port: 11011
      servicePort: 11012
      health:
        ready: true
      tags:
        kuma.io/service: backend
        kuma.io/protocol: http
```

This `health.ready` status is intended to show the status of the endpoint itself.
It is set differently depending on the environment ([Kubernetes](#kubernetes) or [Universal](#universal-probes)).
It's treated the same way regardless of the environment:

- if `health.ready` is true or `health` section is missing - {{site.mesh_product_name}} considers the inbound as healthy and includes it 
  into load balancing set.
- if `health.ready` is false -  {{site.mesh_product_name}} doesn't include the inbound into the load balancing set.

Also, `health.ready` is used to compute the status of the data plane proxy and service. You can see these statuses both in {{site.mesh_product_name}} GUI and {{site.mesh_product_name}} CLI:

Data plane proxy health follows these rules:

- if proxy status is `Offline`, then data plane proxy is `Offline`.
- if proxy status is `Online`:
  - if all inbounds have `health.ready=true` or no `health` then data plane proxy is `Online`
  - if all inbounds have `health.ready=false` then data plane proxy is `Offline`
  - if at least one of the inbounds has `health.ready=false` then data plane proxy is `Partially degraded` 

Service health is computed by aggregating the health of all data plane proxies that have one inbound with a given `kuma.io/service` and is set this way:

- if all data plane proxies are `Online` then the service is `Online`
- if no data plane proxy is `Online` then the service is `Offline`
- if at least one of the data plane proxies is `Offline` are `Partially degraded` then the service is `Partially degraded`

## Kubernetes

{{site.mesh_product_name}} leverages container statuses within the pod to evaluate `dataplane.inbound.health` and will follow these rules:

- If the sidecar container is not ready then all inbound will have `health=false`.
- If the side container is ready then the health of the inbound will be whatever is the readiness of the container which exposes the port used by the inbound. 

{% if_version lte:2.8.x %}
### Kubernetes Probes

Native [probe functionalities](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) are not supported until 2.9 release. `HTTPGet` and `gRPC` probes always fail when mTLS enabled and `TCPSocket` probes always succeed regardless of mTLS.

{% endif_version %}

{% if_version gte:2.9.x %}
### Application Probe Proxy

For better lifecycle management Kubernetes has [readiness, liveness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/). When using {{site.mesh_product_name}}. Application Probe Proxy listens on a non-mTLS port, handles probe requests from Kubernetes and forwards them back to applications. It works by overriding the probe definitions in the Pod so that probe requests will be sent to the proxy.

For example, if we specify the following `HTTPGet` probe:

```yaml
livenessProbe:
  httpGet:
    path: /metrics
    port: 3001
  initialDelaySeconds: 3
  periodSeconds: 3
```

{{site.mesh_product_name}} will replace it with:

```yaml
livenessProbe:
  httpGet:
    path: /3001/metrics
    port: 9001
  initialDelaySeconds: 3
  periodSeconds: 3
```

Similarly, the following `TCPSocket` probe

```yaml
readinessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 3
  periodSeconds: 3
```

will be replaced with:

```yaml
readinessProbe:
  httpGet:
    path: /tcp/5432
    port: 9001
  initialDelaySeconds: 3
  periodSeconds: 3
```

Where `9001` is a default port that Application Probe Proxy listens on. To prevent potential conflicts with applications, you may configure this port using one of these methods:

{% tabs %}
{% tab config-probe-proxy control plane config %}
With the config yaml:
```yaml
runtime:
  kubernetes:
    injector:
      applicationProbeProxyPort: 19100
```

or environment variable: `KUMA_RUNTIME_KUBERNETES_APPLICATION_PROBE_PROXY_PORT=19100`
{% endtab %}
{% tab config-probe-proxy Pod annotation %}
With the Pod annotation:
```yaml
annotations:
  kuma.io/application-probe-proxy-port: 19100
```
{% endtab %}
{% endtabs %}

You can disable Application Probe Proxy by set the port to `0`. When it is disabled, Virtual Probes still works as usual before being removed.

{% endif_version %}

### Virtual Probes

{% if_version gte:2.9.x %}
{% warning %}
Starting with {{site.mesh_product_name}} version 2.9.0, the Virtual Probes feature is deprecated and suppressed. Application Probe Proxy is the successor and will be enabled by default. 
{% endwarning %}
{% endif_version %}

For better lifecycle management Kubernetes has [readiness, liveness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

When using mTLS the container ports are not available outside the pod and Kubernetes will fail to check the probes.
To work around this {{site.mesh_product_name}} generates a special non-mTLs listener and overrides the probe definitions in the Pod to proxy them through this sidecar listener.
These are called *Virtual probes*.

{% warning %}
This feature currently only supports `httpGet` probes.
{% endwarning %}

For example, if we specify the following probe:

```yaml
livenessProbe:
  httpGet:
    path: /metrics
    port: 3001
  initialDelaySeconds: 3
  periodSeconds: 3
```

{{site.mesh_product_name}} will replace it with:

```yaml
livenessProbe:
  httpGet:
    path: /3001/metrics
    port: 9000
  initialDelaySeconds: 3
  periodSeconds: 3
```

Where `9000` is a default virtual probe port, this is configurable:

{% tabs %}
{% tab usage control plane config %}
With the config yaml:
```yaml
runtime:
  kubernetes:
    injector:
      virtualProbesPort: 19001
```

or environment variable: `KUMA_RUNTIME_KUBERNETES_VIRTUAL_PROBES_PORT=19001`
{% endtab %}
{% tab usage Pod annotation %}
With the Pod annotation:
```yaml
annotations:
  kuma.io/virtual-probes-port: 19001
```
{% endtab %}
{% endtabs %}

You can also disable {{site.mesh_product_name}}'s probe virtualization:

{% tabs %}
{% tab probe-visualization control plane config %}

With the config yaml:
```yaml
runtime:
  kubernetes:
    injector:
      virtualProbesEnabled: false
```
or environment variable: `KUMA_RUNTIME_KUBERNETES_VIRTUAL_PROBES_ENABLED=false`
{% endtab %}
{% tab probe-visualization Pod annotation %}
With the Pod annotation:
```yaml
annotations:
  kuma.io/virtual-probes: disabled
```
{% endtab %}
{% endtabs %}

## Universal probes

On Universal there is no single standard for probing the service.
For health checking, the `Dataplane` status on Universal {{site.mesh_product_name}} uses Envoy's Health Discovery Service (HDS).
Envoy does health checks and reports the status back to the {{site.mesh_product_name}} Control Plane.

In order to configure health checks for your `Dataplane` you have to update `inbound` config with `serviceProbe`:

```yaml
type: Dataplane
mesh: default
name: web-01
networking:
  address: 127.0.0.1
  inbound:
    - port: 11011
      servicePort: 11012
      serviceProbe:
        timeout: 2s # optional (default value is taken from KUMA_DP_SERVER_HDS_CHECK_TIMEOUT)
        interval: 1s # optional (default value is taken from KUMA_DP_SERVER_HDS_CHECK_INTERVAL)
        healthyThreshold: 1 # optional (default value is taken from KUMA_DP_SERVER_HDS_CHECK_HEALTHY_THRESHOLD)
        unhealthyThreshold: 1 # optional (default value is taken from KUMA_DP_SERVER_HDS_CHECK_UNHEALTHY_THRESHOLD)
        tcp: {}
      tags:
        kuma.io/service: backend
        kuma.io/protocol: http
```

If there is a `serviceProbe` configured for the inbound, {{site.mesh_product_name}} will automatically fill the `inbound.health` section and update it 
with the interval equal to `KUMA_DP_SERVER_HDS_REFRESH_INTERVAL`.
Alternatively, it's possible to omit a `serviceProbe` section and develop custom automation that periodically updates the `health` of the inbound.

If the grpc stream with Envoy is disconnected then {{site.mesh_product_name}} considers this proxy offline.
It will, however, still advertise inbounds using the final update on their health before disconnection.
This is to avoid connection issues between `kuma-cp` and `kuma-dp` blocking data plane traffic.

Additionally, when `serviceProbe` is defined, probes takes into account a health of Envoy.
When kuma-dp receives the first shutdown signal, it goes into draining state and all inbounds are considered unhealthy.
