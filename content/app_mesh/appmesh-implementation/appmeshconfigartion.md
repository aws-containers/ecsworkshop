+++
title = "App Mesh Configuration"
description = "App Mesh deployment"
weight = 1
+++


**AWS App Mesh** is a service mesh based on the Envoy proxy. It standardizes how microservices communicate, giving you end-to-end visibility and helping to ensure high-availability for your applications. 

hroughout this workshop we will use AWS App Mesh, to gain consistent visibility and network traffic controls on our microservices running in **Amazon EC2 with Fargate**.

A service mesh is composed of two parts, a _control plane_ and a _data plane_. AWS App Mesh is a managed control plane so you as a user don’t need to install or manage any servers to run it. The data plane for App Mesh is the open source Envoy proxy. **_It is your responsability to add the Envoy proxy as a side car to each microservice you want to expose and manage via App Mesh._**

In the next few sections, you will perform 2 broad set of tasks:

 - You will use the AWS CDK to create and configure the App Mesh resources needed to represent the microservices available in your environment like virtual services and virtual nodes.

 - You will add a docker container running the Envoy proxy to each app where you have microservices running. AWS App Mesh supports EC2, ECS and EKS and here we will show you how to add Envoy in ECS, without modifying your microservice source code. 

 - You will add an Ingress (Virtual Gateway) and Virtual Routers, so external clients to the mesh can interact with the (virtual) services that were already brought into the mesh. 

**_A key feature of Envoy worth mentioning_** is that it exposes a set of APIs. These APIs are leveraged by AWS App Mesh to dynamically configure Envoy’s routing logic, freeing developers from the tedious task of manually updating config files.

Now every time you launch an Envoy enabled microservice, the Envoy proxy will contact the AWS App Mesh Management API to subscribe to resource information for Listeners, Clusters, Routes, and Endpoints. The connection from the Envoy proxy to the AWS App Mesh management API endpoint is held open, which allows AWS App Mesh to stream updates to the Envoy proxy as users introduce changes to either of the resources listed above.

Let’s start by creating the service mesh. A service mesh is a logical boundary for network traffic between the services that resides within it. To enable the app mesh, we will configure the ecsdemo-platform app as follow:

Open the terminal and execute:
```bash
c9 ~/environment/ecsdemo-platform/cdk/app.py
```

Go to line #213 and uncomment `# self.appmesh()` or execute the following command in the terminal:
```bash
sed -i -e '/self.appmesh()/s/# //' ~/environment/ecsdemo-platform/cdk/app.py
```

Install any CDK python prerequisites (libraries) needed by the ecsdemo-platform application 
```bash
pip install --upgrade -r ~/environment/ecsdemo-platform/cdk/requirements.txt 
```

Confirm that the CDK can synthesize the assembly CloudFormation templates
```bash
cd ~/environment/ecsdemo-platform/cdk
cdk synth
```

View proposed changes to the environment
```bash
cdk diff
```

Deploy the changes to the environment:
```bash
cdk deploy --require-approval never
```

{{% notice info %}}
The information we are going to review moving forward is focused solely on the App Mesh implementations. For explanations regarding ECS CDK implementation, please, check out the workshop "[Deploying Microservices to ECS]({{< ref "microservices/" >}})"
{{% /notice %}}

Let’s take a look at what’s being built from the App mesh perspective. You may notice that everything defined in the stack is 100% written as python code.
    

```python
# This will create the app mesh (control plane)
self.mesh = aws_appmesh.Mesh(self,"EcsWorkShop-AppMesh", mesh_name="ecs-mesh")

# We will create a App Mesh Virtual Gateway
self.mesh_vgw = aws_appmesh.VirtualGateway(
    self,
    "Mesh-VGW",
    mesh=self.mesh,
    listeners=[aws_appmesh.VirtualGatewayListener.http(
        port=3000
        )],
    virtual_gateway_name="ecsworkshop-vgw"
)

# Creating the App Mesh Envoy Proxy task 
# For more info related to App Mesh Proxy check https://docs.aws.amazon.com/app-mesh/latest/userguide/getting-started-ecs.html
self.mesh_gw_proxy_task_def = aws_ecs.FargateTaskDefinition(
    self,
    "mesh-gw-proxy-taskdef",
    cpu=256,
    memory_limit_mib=512,
    family="mesh-gw-proxy-taskdef",
)

# LogGroup for the App Mesh Proxy Task
self.logGroup = aws_logs.LogGroup(self,"ecsworkshopMeshGateway",    
    retention=aws_logs.RetentionDays.ONE_WEEK
)

# App Mesh Virtual Gateway Envoy proxy Task definition
# For a use specific ECR region, please check https://docs.aws.amazon.com/app-mesh/latest/userguide/envoy.html
self.container = self.mesh_gw_proxy_task_def.add_container(
    "mesh-gw-proxy-contdef",
    image=aws_ecs.ContainerImage.from_registry("public.ecr.aws/appmesh/aws-appmesh-envoy:v1.18.3.0-prod"),
    container_name="envoy",
    memory_reservation_mib=512,
    environment={
        "REGION": getenv('AWS_DEFAULT_REGION'),
        "ENVOY_LOG_LEVEL": "debug",
        "ENABLE_ENVOY_STATS_TAGS": "1",
        # "ENABLE_ENVOY_XRAY_TRACING": "1",
        "APPMESH_RESOURCE_ARN": self.mesh_vgw.virtual_gateway_arn
    },
    essential=True,
    logging=aws_ecs.LogDriver.aws_logs(
        stream_prefix='/mesh-gateway',
        log_group=self.logGroup
    ),
    health_check=aws_ecs.HealthCheck(
        command=["CMD-SHELL","curl -s http://localhost:9901/server_info | grep state | grep -q LIVE"],
    )
)

# Default port where our front end application is listening
self.container.add_port_mappings(
    aws_ecs.PortMapping(
        container_port=3000
    )
)


# For environment variables check https://docs.aws.amazon.com/app-mesh/latest/userguide/envoy-config.html
self.mesh_gateway_proxy_fargate_service = aws_ecs_patterns.NetworkLoadBalancedFargateService(
    self,
    "MeshGW-Proxy-Fargate-Service",
    service_name='mesh-gw-proxy',
    cpu=256,
    memory_limit_mib=512,
    desired_count=1,
    listener_port=80,
    assign_public_ip=True,
    task_definition=self.mesh_gw_proxy_task_def,
    cluster=self.ecs_cluster,
    public_load_balancer=True,
    cloud_map_options=aws_ecs.CloudMapOptions(
        cloud_map_namespace=self.ecs_cluster.default_cloud_map_namespace,
        name='mesh-gw-proxy'
    )
    # deployment_controller,
    # circuit_breaker,
)

# For testing purposes we will open any ipv4 requests to port 3000
self.mesh_gateway_proxy_fargate_service.service.connections.allow_from_any_ipv4(
    port_range=aws_ec2.Port(protocol=aws_ec2.Protocol.TCP, string_representation="vtw_proxy", from_port=3000, to_port=3000),
    description="Allow NLB connections on port 3000"
)

self.mesh_gw_proxy_task_def.default_container.add_ulimits(aws_ecs.Ulimit(
    hard_limit=15000,
    name=aws_ecs.UlimitName.NOFILE,
    soft_limit=15000
    )
)

#Adding necessary policies for Envoy proxy to communicate with required services
self.mesh_gw_proxy_task_def.execution_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEC2ContainerRegistryReadOnly"))
self.mesh_gw_proxy_task_def.execution_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("CloudWatchLogsFullAccess"))

self.mesh_gw_proxy_task_def.task_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("CloudWatchFullAccess"))
# self.mesh_gw_proxy_task_def.task_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("AWSXRayDaemonWriteAccess"))
self.mesh_gw_proxy_task_def.task_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("AWSAppMeshEnvoyAccess"))

self.mesh_gw_proxy_task_def.execution_role.add_to_policy(
    aws_iam.PolicyStatement(
        actions=['ec2:DescribeSubnets'],
        resources=['*']
    )
)

core.CfnOutput(self, "MeshGwNlbDns",value=self.mesh_gateway_proxy_fargate_service.load_balancer.load_balancer_dns_name,export_name="MeshGwNlbDns")
core.CfnOutput(self, "MeshArn",value=self.mesh.mesh_arn,export_name="MeshArn")
core.CfnOutput(self, "MeshName",value=self.mesh.mesh_name,export_name="MeshName")
core.CfnOutput(self, "MeshEnvoyServiceArn",value=self.mesh_gateway_proxy_fargate_service.service.service_arn,export_name="MeshEnvoyServiceArn")
core.CfnOutput(self, "MeshVGWArn",value=self.mesh_vgw.virtual_gateway_arn,export_name="MeshVGWArn")
core.CfnOutput(self, "MeshVGWName",value=self.mesh_vgw.virtual_gateway_name,export_name="MeshVGWName")

```

When the stack is done building, it will print out all of the outputs for the underlying CloudFormation stack. These outputs are what we use to reference the base platform when deploying the microservices. Below is an example of what the outputs look like:

```bash
Outputs:
ecsworkshop-base.ECSClusterName = container-demo
ecsworkshop-base.ECSClusterSecGrp = []
ecsworkshop-base.FE2BESecGrp = sg-079051e73bd142cd7
ecsworkshop-base.MeshArn = arn:aws:appmesh:us-west-2:875448814018:mesh/ecs-mesh
ecsworkshop-base.MeshEnvoyServiceArn = arn:aws:ecs:us-west-2:875448814018:service/container-demo/mesh-gw-proxy
ecsworkshop-base.MeshGWProxyFargateServiceLoadBalancerDNS887B20A3 = ecswo-MeshG-15QP3AYEMS3HX-e9cb6dc2be359121.elb.us-west-2.amazonaws.com
ecsworkshop-base.MeshGwNlbDns = ecswo-MeshG-15QP3AYEMS3HX-e9cb6dc2be359121.elb.us-west-2.amazonaws.com
ecsworkshop-base.MeshName = ecs-mesh
ecsworkshop-base.MeshVGWArn = arn:aws:appmesh:us-west-2:875448814018:mesh/ecs-mesh/virtualGateway/ecsworkshop-vgw
ecsworkshop-base.MeshVGWName = ecsworkshop-vgw
ecsworkshop-base.NSArn = arn:aws:servicediscovery:us-west-2:875448814018:namespace/ns-rychmmvjc2sbpj3g
ecsworkshop-base.NSId = ns-rychmmvjc2sbpj3g
ecsworkshop-base.NSName = service.local
ecsworkshop-base.ServicesSecGrp = sg-079051e73bd142cd7
ecsworkshop-base.StressToolEc2Id = i-057a3558a86f6bed2
ecsworkshop-base.StressToolEc2Ip = 10.0.0.112
```


The following resources have been created:

- [Base Platform]({{< ref "/microservices/platform/build_environment" >}})
- NLB: Instead of using an ALB, we will deploy an NLB since we the routing configurations will be done using the Virtual Gateway
- App Mesh Virtual Gateway: A virtual gateway allows resources that are outside of your mesh to communicate to resources that are inside of your mesh. The virtual gateway represents an Envoy proxy running in an Amazon ECS service, in a Kubernetes service, or on an Amazon EC2 instance. Unlike a virtual node, which represents Envoy running with an application, a virtual gateway represents Envoy deployed by itself.
    - External resources must be able to resolve a DNS name to an IP address assigned to the service or instance that runs Envoy, in this case we will use CloudMap. Envoy can then access all of the App Mesh configuration for resources that are inside of the mesh. The configuration for handling the incoming requests at the Virtual Gateway are specified using Gateway Routes.

This is an abstract architecture of what we have deployed so far:

![infra-nodejs](../images/ecs-app-mesh-diagram-Infra-Mesh.png)