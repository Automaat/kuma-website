---
title: Explore Kuma with the Universal demo app
---

To start learning how {{site.mesh_product_name}} works, you can download and run a simple demo application that consists of two services:

- `demo-app`: web application that lets you increment a numeric counter
- `redis`: data store for the counter

This guide also introduces some of the tools {{site.mesh_product_name}} provides to help you control and monitor traffic, track resource status, and more.

The `demo-app` service listens on port 5000. When it starts, it expects to find a zone key in Redis that specifies the name of the datacenter (or cluster) where the Redis instance is running. This name is displayed in the browser.

The zone key is purely static and arbitrary. Different zone values for different Redis instances let you keep track of which Redis instance stores the counter if you manage routes across different zones, clusters, and clouds.

## Prerequisites

- [Redis installed](https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack)
- [{{site.mesh_product_name}} installed](/install)
- [Demo app downloaded from GitHub](https://github.com/kumahq/kuma-counter-demo):

  ```sh
  git clone https://github.com/kumahq/kuma-counter-demo.git
  ```

To explore traffic metrics with the demo app, you also need to [set up Prometheus](https://prometheus.io/docs/prometheus/latest/getting_started/). See the [traffic metrics policy documentation](/docs/{{ page.release }}/policies/traffic-metrics).

## Set up

1.  Run `redis` as a daemon on port 26379 and set a default zone name:

    ```sh
    redis-server --port 26379 --daemonize yes
    redis-cli -p 26379 set zone local
    ```

1.  Install and start `demo-app` on the default port 5000:

    ```sh
    npm install --prefix=app/
    npm start --prefix=app/
    ```

## Generate tokens

Create a token for Redis and a token for the app (all valid for 30 days):

```sh
kumactl generate dataplane-token --tag kuma.io/service=redis --valid-for=720h > kuma-token-redis
kumactl generate dataplane-token --tag kuma.io/service=app --valid-for=720h > kuma-token-app
```

{% warning %}
This action requires [authentication](/docs/{{ page.release }}/production/secure-deployment/api-server-auth/#admin-user-token) unless executed against a control-plane running on localhost.
If `kuma-cp` is running inside docker container please see [docker authentication docs](/docs/{{ page.release }}/production/cp-deployment/stand-alone/).
{% endwarning %}

## Create a data plane proxy for each service

For Redis:

```sh
kuma-dp run \
  --cp-address=https://localhost:5678/ \
  --dns-enabled=false \
  --dataplane-token-file=kuma-token-redis \
  --dataplane="
  type: Dataplane
  mesh: default
  name: redis
  networking: 
    address: 127.0.0.1
    inbound: 
      - port: 16379
        servicePort: 26379
        serviceAddress: 127.0.0.1
        tags: 
          kuma.io/service: redis
          kuma.io/protocol: tcp
    admin:
      port: 9901"
```

And for the demo app:

```sh
kuma-dp run \
  --cp-address=https://localhost:5678/ \
  --dns-enabled=false \
  --dataplane-token-file=kuma-token-app \
  --dataplane="
  type: Dataplane
  mesh: default
  name: app
  networking: 
    address: 127.0.0.1
    outbound:
      - port: 6379
        tags:
          kuma.io/service: redis
    inbound: 
      - port: 15000
        servicePort: 5000
        serviceAddress: 127.0.0.1
        tags: 
          kuma.io/service: app
          kuma.io/protocol: http
    admin:
      port: 9902"
```

## Run

Navigate to 127.0.0.1:5000 and increment the counter.

## Explore the mesh

You can view the sidecar proxies that are connected to the {{site.mesh_product_name}} control plane:

{% tabs %}
{% tab usage GUI (Read-Only) %}

{{site.mesh_product_name}} ships with a **read-only** GUI that you can use to retrieve {{site.mesh_product_name}} resources. By default the GUI listens on the API port and defaults to `:5681/gui`.

You can navigate to [`127.0.0.1:5681/meshes/default/dataplanes`](http://127.0.0.1:5681/meshes/default/dataplanes) to see the connected dataplanes.

{% endtab %}
{% tab usage HTTP API (Read/Write) %}

{{site.mesh_product_name}} ships with a **read-only** HTTP API that you can use to retrieve {{site.mesh_product_name}} resources. 

By default the HTTP API listens on port `5681`. 

Navigate to [`127.0.0.1:5681/meshes/default/dataplanes`](http://127.0.0.1:5681/meshes/default/dataplanes) to see the connected dataplanes.

{% endtab %}
{% tab usage kumactl (Read/Write) %}

You can use the `kumactl` CLI to perform **read-only** operations on {{site.mesh_product_name}} resources. The `kumactl` binary is a client to the {{site.mesh_product_name}} HTTP API, you will need to first port-forward the API service with:

Run `kumactl`, for example:

```sh
kumactl get dataplanes
# MESH      NAME                                              TAGS
# default   kuma-demo-app-68758d8d5d-dddvg.kuma-demo          app=kuma-demo-demo-app env=prod pod-template-hash=68758d8d5d protocol=http service=demo-app_kuma-demo_svc_5000 version=v8
# default   redis-master-657c58c859-5wkb4.kuma-demo           app=redis pod-template-hash=657c58c859 protocol=tcp role=master service=redis_kuma-demo_svc_6379 tier=backend
```

You can configure `kumactl` to point to any zone `kuma-cp` instance by running:

```sh
kumactl config control-planes add --name=XYZ --address=http://{address-to-kuma}:5681
```
{% endtab %}
{% endtabs %}

## Enable Mutual TLS and Traffic Permissions

By default the network is unsecure and not encrypted. We can change this with {{site.mesh_product_name}} by enabling the [Mutual TLS](/docs/{{ page.release }}/policies/mutual-tls/) policy to provision a dynamic Certificate Authority (CA) on the `default` [Mesh](/docs/{{ page.release }}/production/mesh/) resource that will automatically assign TLS certificates to our services (more specifically to the injected dataplane proxies running alongside the services).

We can enable Mutual TLS with a `builtin` CA backend by executing:

```sh
cat <<EOF | kumactl apply -f -
type: Mesh
name: default
mtls:
  enabledBackend: ca-1
  backends:
  - name: ca-1
    type: builtin
EOF
```

Once Mutual TLS has been enabled, {{site.mesh_product_name}} will **not allow** traffic to flow freely across our services unless we explicitly have a [Traffic Permission](/docs/{{ page.release }}/policies/traffic-permissions/) policy that describes what services can be consumed by other services.
By default, a very permissive traffic permission is created.

For the sake of this demo we will delete it:

```sh
kumactl delete traffic-permission allow-all-default
```

You can try to make requests to the demo application at [`127.0.0.1:5000/`](http://127.0.0.1:5000/) and you will notice that they will **not** work.

Now let's add back the default traffic permission:
```sh
cat <<EOF | kumactl apply -f -
type: TrafficPermission
name: allow-all-default
mesh: default
sources:
  - match:
      kuma.io/service: '*'
destinations:
  - match:
      kuma.io/service: '*'
EOF
```

By doing so every request we now make on our demo application at [`127.0.0.1:5000/`](http://127.0.0.1:5000/) is not only working again, but it is automatically encrypted and secure.

{% tip %}
As usual, you can visualize the Mutual TLS configuration and the Traffic Permission policies we have just applied via the GUI, the HTTP API or `kumactl`.
{% endtip %}

## Explore Traffic Metrics

One of the most important [policies](/policies) that {{site.mesh_product_name}} provides out of the box is [Traffic Metrics](/docs/{{ page.release }}/policies/traffic-metrics/).

With Traffic Metrics we can leverage Prometheus and Grafana to provide powerful dashboards that visualize the overall traffic activity of our application and the status of the service mesh.

{% if_version lte:2.3.x %}
```sh
cat <<EOF | kumactl apply -f -
type: Mesh
name: default
mtls:
  enabledBackend: ca-1
  backends:
  - name: ca-1
    type: builtin
metrics:
  enabledBackend: prometheus-1
  backends:
  - name: prometheus-1
    type: prometheus
    conf:
      skipMTLS: true
EOF
```
{% endif_version %}
{% if_version gte:2.4.x %}
```sh
cat <<EOF | kumactl apply -f -
type: Mesh
name: default
mtls:
  enabledBackend: ca-1
  backends:
  - name: ca-1
    type: builtin
metrics:
  enabledBackend: prometheus-1
  backends:
  - name: prometheus-1
    type: prometheus
    conf:
      tls:
        mode: disabled
EOF
```
{% endif_version %}

This will enable the `prometheus` metrics backend on the `default` [Mesh](/docs/{{ page.release }}/production/mesh/) and automatically collect metrics for all of our traffic.

Increment the counter to generate traffic, and access the dashboard at [127.0.0.1:3000](http://127.0.0.1:3000) with default credentials for both the username (`admin`) and the password (`admin`).

{{site.mesh_product_name}} automatically installs three dashboard that are ready to use:

* `{{site.mesh_product_name}} Mesh`: to visualize the status of the overall Mesh.
* `{{site.mesh_product_name}} Dataplane`: to visualize metrics for a single individual dataplane.
* `{{site.mesh_product_name}} Service to Service`: to visualize traffic metrics for our services.

You can now explore the dashboards and see the metrics being populated over time.

## Next steps

* Explore the [Policies](/policies) available to govern and orchestrate your service traffic.
* Read the [full documentation](/docs/{{ page.release }}/) to learn about all the capabilities of {{site.mesh_product_name}}.
* Chat with us at the official [Kuma Slack](/community) for questions or feedback.
