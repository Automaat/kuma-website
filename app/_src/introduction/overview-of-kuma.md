---
title: Overview of Kuma
---

{{site.mesh_product_name}} is a platform agnostic open-source control plane for service mesh and microservices management, with support for Kubernetes, VM, and bare metal environments.

<center>
<img src="/assets/images/diagrams/main-diagram@2x.png" alt="" style="width: 550px; padding-top: 10px"/>
</center>

{{site.mesh_product_name}} helps implement a service mesh approach to distributed deployments as part of the move from monolithic architectures to microservices. You can run a service mesh with {{site.mesh_product_name}} before you start decomposing your monolith, which helps keep your network secure and observable as your architecture changes. {{site.mesh_product_name}} is:

{% if_version gte:2.6.x %}
* **Universal and Kubernetes-native**: Platform-agnostic, can run and operate anywhere.
* **Single-zone and multi-zone**: Supports multiple clouds, regions, and Kubernetes clusters with native DNS service discovery and ingress capability.
  * [Read more about single-zone deployments](/docs/{{ page.release }}/production/deployment/single-zone/)
  * [Read more about multi-zone deployments](/docs/{{ page.release }}/production/deployment/multi-zone/)
* **Multi-mesh**: Supports multiple individual meshes with one control plane, lowering the operational costs of supporting the entire organization.
* **Attribute-based policies**: Let you apply fine grained service and traffic policies with any arbitrary tag selector for `sources` and `destinations`.
* **Envoy-based**: Powered by Envoy sidecar proxies, without exposing the complexity of Envoy itself.
* **Horizontally scalable**
* **Enterprise-ready**: Supports mission critical enterprise use cases that require uptime and stability.
{% endif_version %}

{% if_version lte:2.5.x %}
* **Universal and Kubernetes-native**: Platform-agnostic, can run and operate anywhere.
* **Standalone and multi-zone**: Supports multiple clouds, regions, and Kubernetes clusters with native DNS service discovery and ingress capability.
  * [Read more about standalone deployments](/docs/{{ page.release }}/production/deployment/stand-alone/)
  * [Read more about multi-zone deployments](/docs/{{ page.release }}/production/deployment/multi-zone/)
* **Multi-mesh**: Supports multiple individual meshes with one control plane, lowering the operational costs of supporting the entire organization.
* **Attribute-based policies**: Let you apply fine grained service and traffic policies with any arbitrary tag selector for `sources` and `destinations`.
* **Envoy-based**: Powered by Envoy sidecar proxies, without exposing the complexity of Envoy itself.
* **Horizontally scalable**
* **Enterprise-ready**: Supports mission critical enterprise use cases that require uptime and stability.
{% endif_version %}

Bundling [Envoy](https://envoyproxy.io/) as the data plane, {{site.mesh_product_name}} can instrument any L4/L7 traffic to secure, observe, route and enhance connectivity between any services or databases. It can be used natively in Kubernetes via CRDs or via a RESTful API across other environments.

Example of a multi-zone deployment for multiple Kubernetes clusters, or a hybrid Kubernetes/VM cluster:

<center>
<img src="/assets/images/diagrams/gslides/kuma_multizone.svg" alt="Kuma service mesh multi zone deployment" style="padding-top: 20px; padding-bottom: 10px;"/>
</center>

The core maintainer of Kuma is **Kong**, the maker of the popular open-source Kong Gateway 🦍.
