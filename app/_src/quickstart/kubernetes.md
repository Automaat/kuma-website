---
title: Explore Kuma with the Kubernetes demo app
---

To start learning how {{site.mesh_product_name}} works, you can download and run a simple demo application that consists of two services:

- `demo-app`: web application that lets you increment a numeric counter
- `redis`: data store for the counter

This guide also introduces some of the tools {{site.mesh_product_name}} provides to help you control and monitor traffic, track resource status, and more.

The `demo-app` service listens on port 5000. When it starts, it expects to find a zone key in Redis that specifies the name of the datacenter (or cluster) where the Redis instance is running. This name is displayed in the browser.

The zone key is purely static and arbitrary. Different zone values for different Redis instances let you keep track of which Redis instance stores the counter if you manage routes across different zones, clusters, and clouds.

## Prerequisites

- 
[{{site.mesh_product_name}} installed on your Kubernetes cluster](/docs/{{ page.release }}/production/deployment/stand-alone/)
- [Demo app downloaded from GitHub](https://github.com/kumahq/kuma-counter-demo):

  ```sh
  git clone https://github.com/kumahq/kuma-counter-demo.git
  ```

## Set up and run

Two different YAML files are available:

- `demo.yaml` installs the basic resources
- `demo-v2.yaml` installs the frontend service with different colors. This lets you more clearly view routing across multiple versions, for example.
- [`gateway.yaml` installs a builtin gateway](#builtin-gateways)

1.  Install resources in a `kuma-demo` namespace:

    ```sh
    kubectl apply -f demo.yaml
    ```

1.  Port forward the service to the namespace on port 5000:

    ```sh
    kubectl port-forward svc/demo-app -n kuma-demo 5000:5000
    ```

1.  In a browser, go to `127.0.0.1:5000` and increment the counter.

## Explore the mesh

The demo app includes the `kuma.io/sidecar-injection` label enabled on the `kuma-demo` namespace. This means that {{site.mesh_product_name}} [already knows](/docs/{{ page.release }}/production/dp-config/dpp-on-kubernetes/) that it needs to automatically inject a sidecar proxy to every Kubernetes deployment in the `default` [Mesh](/docs/{{ page.release }}/production/mesh/) resource:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kuma-demo
  labels:
    kuma.io/sidecar-injection: enabled
```

You can view the sidecar proxies that are connected to the {{site.mesh_product_name}} control plane:

{% tabs %}
{% tab usage GUI (Read-Only) %}

{{site.mesh_product_name}} ships with a **read-only** GUI that you can use to retrieve {{site.mesh_product_name}} resources. By default the GUI listens on the API port and defaults to `:5681/gui`. 

To access {{site.mesh_product_name}} we need to first port-forward the API service with:

```sh
kubectl port-forward svc/{{site.mesh_cp_name}} -n {{site.mesh_namespace}} 5681:5681
```

And then navigate to [`127.0.0.1:5681/gui`](http://127.0.0.1:5681/gui) to see the GUI. 

{% endtab %}
{% tab usage HTTP API (Read-Only) %}

{{site.mesh_product_name}} ships with a **read-only** HTTP API that you can use to retrieve {{site.mesh_product_name}} resources. 

By default the HTTP API listens on port `5681`. To access {{site.mesh_product_name}} we need to first port-forward the API service with:

```sh
kubectl port-forward svc/{{site.mesh_cp_name}} -n {{site.mesh_namespace}} 5681:5681
```

And then you can navigate to [`127.0.0.1:5681/meshes/default/dataplanes`](http://127.0.0.1:5681/meshes/default/dataplanes) to see the connected dataplanes.

{% endtab %}
{% tab usage kumactl (Read-Only) %}

You can use the `kumactl` CLI to perform **read-only** operations on {{site.mesh_product_name}} resources. The `kumactl` binary is a client to the {{site.mesh_product_name}} HTTP API, you will need to first port-forward the API service with:

```sh
kubectl port-forward svc/{{site.mesh_cp_name}} -n {{site.mesh_namespace}} 5681:5681
```

and then run `kumactl`, for example:

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

By default, the network is unsecure and not encrypted. We can change this with {{site.mesh_product_name}} by enabling the [Mutual TLS](/docs/{{ page.release }}/policies/mutual-tls/) policy to provision a dynamic Certificate Authority (CA) on the `default` [Mesh](/docs/{{ page.release }}/production/mesh/) resource that will automatically assign TLS certificates to our services (more specifically to the injected dataplane proxies running alongside the services).

We can enable Mutual TLS with a `builtin` CA backend by executing:

```sh
echo "apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
    - name: ca-1
      type: builtin" | kubectl apply -f -
```

Once Mutual TLS has been enabled, {{site.mesh_product_name}} will **not allow** traffic to flow freely across our services unless we explicitly have a [Traffic Permission](/docs/{{ page.release }}/policies/traffic-permissions/) policy that describes what services can be consumed by other services.
By default, a very permissive traffic permission is created.

For the sake of this demo we will delete it:

```sh
kubectl delete trafficpermission allow-all-default
```

You can try to make requests to the demo application at [`127.0.0.1:5000/`](http://127.0.0.1:5000/) and you will notice that they will **not** work.

Now let's add back the default traffic permission:

```sh
echo "apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  name: allow-all-default
spec:
  sources:
    - match:
        kuma.io/service: '*'
  destinations:
    - match:
        kuma.io/service: '*'" | kubectl apply -f -
```

By doing so every request we now make on our demo application at [`127.0.0.1:5000/`](http://127.0.0.1:5000/) is not only working again, but it is automatically encrypted and secure.

{% tip %}
As usual, you can visualize the Mutual TLS configuration and the Traffic Permission policies we have just applied via the GUI, the HTTP API or `kumactl`.
{% endtip %}

## Builtin gateways

The resources for creating a builtin gateway is included with
`kuma-counter-demo` in `gateway.yaml` as well:

- a `MeshGateway` that sets up the listeners
- a `MeshGatewayRoute` that directs requests to the demo app
- a `MeshGatewayInstance` that manages and deploys proxies to serve gateway
  traffic

Learn more about builtin gateways in [the dedicated gateway
docs.](/docs/{{ page.release }}/explore/gateway/#builtin)

## Explore Observability features

With `kumactl` you can quickly install all observability components (metrics, logs, tracing) with a single command:

```sh
kumactl install observability | kubectl apply -f -
```

Once that is installed you can use different policies to configure each component. 

### Traffic Metrics

One of the most important [policies](/policies) that {{site.mesh_product_name}} provides out of the box is [Traffic Metrics](/docs/{{ page.release }}/policies/traffic-metrics/).

With Traffic Metrics we can leverage Prometheus and Grafana to provide powerful dashboards that visualize the overall traffic activity of our application and the status of the service mesh.

We can now go ahead and enable metrics on our [Mesh]() object by executing:

```sh
echo "apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
    - name: ca-1
      type: builtin
  metrics:
    enabledBackend: prometheus-1
    backends:
    - name: prometheus-1
      type: prometheus" | kubectl apply -f -
```

This will enable the `prometheus` metrics backend on the `default` [Mesh](/docs/{{ page.release }}/production/mesh/) and automatically collect metrics for all of our traffic.

Increment the counter to generate traffic. Then you can expose the Grafana dashboard:

```sh
kubectl port-forward svc/grafana -n mesh-observability 3000:80
```

and access the dashboard at [127.0.0.1:3000](http://127.0.0.1:3000) with default credentials for both the username (`admin`) and the password (`admin`).

{{site.mesh_product_name}} automatically installs three dashboard that are ready to use:

* `{{site.mesh_product_name}} Mesh`: to visualize the status of the overall Mesh.
* `{{site.mesh_product_name}} Dataplane`: to visualize metrics for a single individual dataplane.
* `{{site.mesh_product_name}} Service to Service`: to visualize traffic metrics for our services.

You can now explore the dashboards and see the metrics being populated over time.

### Traffic logs and trace

You can check out specific instructions on [Traffic Log](/docs/{{ page.release }}/policies/traffic-log) and [Traffic Trace](/docs/{{ page.release }}/policies/traffic-trace) policies in separate documents.

## Next steps

* Explore the [Policies](/policies) available to govern and orchestrate your service traffic.
* Read the [full documentation](/docs/{{ page.release }}/) to learn about all the capabilities of {{site.mesh_product_name}}.
* Chat with us at the official [Kuma Slack](/community) for questions or feedback.
