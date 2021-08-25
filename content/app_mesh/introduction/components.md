+++
title = "Components And How It Works"
description = "Components And How It Works"
weight = 2
+++

## Components

**Service** mesh – A service mesh is a logical boundary for network traffic between the services that reside within it.

**Virtual services** – A virtual service is an abstraction of a real service that is provided by a virtual node directly or indirectly by means of a virtual router.

**Virtual nodes** – A virtual node acts as a logical pointer to a particular task group, such as an ECS service or a Kubernetes deployment. When you create a virtual node, you must specify the service discovery name for your task group.

**Envoy proxy** – The Envoy proxy configures your microservice task group to use the App Mesh service mesh traffic rules that you set up for your virtual routers and virtual nodes. You add the Envoy container to your task group after you have created your virtual nodes, virtual routers, routes, and virtual services.

**Virtual routers** – The virtual router handles traffic for one or more virtual services within your mesh.

**Routes** – A route is associated with a virtual router, and it directs traffic that matches a service name prefix to one or more virtual nodes.

**Virtual GateWays** - A virtual gateway allows resources that are outside of your mesh to communicate to resources that are inside of your mesh. The virtual gateway represents an Envoy proxy running in an Amazon ECS service, in a Kubernetes service, or on an Amazon EC2 instance. Unlike a virtual node, which represents Envoy running with an application, a virtual gateway represents Envoy deployed by itself.

- **Gateway routes** - This is the configuration for handling the incoming requests at the Virtual Gateway. A gateway route is attached to a virtual gateway and routes traffic to an existing virtual service. If a route matches a request, it can distribute traffic to a target virtual service. This topic helps you work with gateway routes in a service mesh.


## How it Works

Now let's check out the following image, in this configuration, the services no longer communicate with each other directly. Instead, they communicate with each other through a proxy. The proxy deployed with the servicea.apps.local service reads the App Mesh configuration and sends traffic to serviceb.apps.local or servicebv2.apps.local, based on the configuration.

![app-mesh-proxy](../images/simple-app-with-mesh-diagram.png)


**Before App Mesh** - Communications and monitoring are manually configured for every service.

![app-mesh-before](../images/before_appmesh.png)

**After App Mesh** - App Mesh configures communications and monitoring for all services.

![app-mesh-after](../images/after_appmesh.png)
