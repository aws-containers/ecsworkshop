+++
title = "What is App Mesh"
description = "Overview"
weight = 1
+++

![app-mesh-proxy](../images/appmesh-proxy.png)

AWS App Mesh is a service mesh that provides application-level networking to make it easy for your services to communicate with each other across multiple types of compute infrastructure. App Mesh standardizes how your services communicate, giving you end-to-end visibility and ensuring high-availability for your applications.

Modern applications are typically composed of multiple services. Each service may be built using multiple types of compute infrastructure such as Amazon EC2, Amazon EKS and AWS Fargate. As the number of services grow within an application, it becomes difficult to pinpoint the exact location of errors, re-route traffic after failures, and safely deploy code changes. Previously, this has required you to build monitoring and control logic directly into your code and redeploy your service every time there are changes.

AWS App Mesh makes it easy to run services by providing consistent visibility and network traffic controls for services built across multiple types of compute infrastructure. App Mesh removes the need to update application code to change how monitoring data is collected or traffic is routed between services. App Mesh configures each service to export monitoring data and implements consistent communications control logic across your application. This makes it easy to quickly pinpoint the exact location of errors and automatically re-route network traffic when there are failures or when code changes need to be deployed.

You can use App Mesh with AWS Fargate, Amazon EC2, Amazon ECS, Amazon EKS, and Kubernetes running on AWS, to better run your application at scale. App Mesh uses the open source Envoy proxy, making it compatible with a wide range of AWS partner and open source tools.

More information on what App Mesh is all about can be found on the official App Mesh documentation page.