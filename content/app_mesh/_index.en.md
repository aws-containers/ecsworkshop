---
title: "App Mesh Integration"
chapter: true
weight: 80
pre: '<i class="fa fa-film" aria-hidden="true"></i> '
---

# Welcome to the AWS App Mesh Workshop!

![app-mesh-architecture](images/welcome-appmesh-architecture-reference.png)

The intent of this workshop is to educate users about the features and usage of AWS App Mesh. Background in ECS and container workflows are not required, but they are recommended. In this chapter I will introduce you to the basic workings of App Mesh, laying the foundation for the hands-on portion of the workshop. Specifically, we will walk you through the following topics:

- Introduction: Which is a general overview of what AWS App Mesh is, its components and how it works
- We will reuse the existing ECS workshop microservices applications and put them behind App Mesh
    - We will implement Virtual Gateways without TLS
    - Highlight tracing and telemetry
    - Implement Mutual TLS
    - Implement BG/Canary deployments


