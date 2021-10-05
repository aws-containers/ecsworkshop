+++
title = "Mesh Frontend Service"
description = "Mesh Frontend Service"
weight = 4
+++

At this point, you should have Crystal and NodeJs backends services running in ECS and their respective App Mesh configurations.

In this chapter, our goal is to edit your ECS Frontend App in order to fully enable request from the exterior of our Mesh to the ECS containers. As depicted in the pictures below:

**Infrastructure setup:**
![infra-setup](../images/ecs-app-mesh-diagram-Infra-setup.png)


### Preparing CDK Code To Deploy App Mesh Resources And ECS Configurations

The Frontend CDK code will change a bit more than previous Crystal and Nodejs files, since we need to utilize previous App Mesh configurations to reference Crystal and NodeJs Virtual Services and Virtual Nodes. 


First, we will comment our new class `FrontendServiceMesh` and comment the previous class `FrontendService` in the file `~/environment/ecsdemo-frontend/cdk/app.py` or run this command in the terminal:
```
#Commenting previous class
sed -i -e '/FrontendService(app, stack_name, env=_env)/s/^#*/#/' ~/environment/ecsdemo-frontend/cdk/app.py 
#Uncommenting new class
sed -i -e "/FrontendServiceMesh(app, stack_name, env=_env)/s/# //" ~/environment/ecsdemo-frontend/cdk/app.py 
```

### Deploying Configurations

Install any CDK python prerequisites (libraries) needed by the ecsdemo-frontend application 
```bash
pip install --upgrade -r ~/environment/ecsdemo-frontend/cdk/requirements.txt 
```

Confirm that the CDK can synthesize the assembly CloudFormation templates
```bash
cd ~/environment/ecsdemo-frontend/cdk
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

Now, to check the application running, you can get the NLB public endpoint from the base platform output or execute the following command:
```bash
echo "http://$(aws cloudformation describe-stacks --stack-name ecsworkshop-base --query "Stacks[0].Outputs[?OutputKey=='MeshGwNlbDns'].OutputValue" --output text)"
```

#### Let's explain the different peaces of the deployment
For this particular service we created a different Class called "FrontendServiceMesh", where we encapsulated all the logic to create the ECS service and the App Mesh resources for the Frontend app, so we can be able to communicate to the Crystal and NodeJs backends and at the same time be able to be reached by the App Mesh Virtual Gateway.  

The first thing we needed to do is to pull Crystal and NodeJs mesh configurations so we can use them when configuring Frontend Virtual Node and the Virtual gateway router. That section is:

```python
self.mesh = aws_appmesh.Mesh.from_mesh_arn(
    self,
    "EcsWorkShop-AppMesh",
    mesh_arn=core.Fn.import_value("MeshArn")
)

self.mesh_vgw = aws_appmesh.VirtualGateway.from_virtual_gateway_attributes(
    self,
    "Mesh-VGW",
    mesh=self.mesh,
    virtual_gateway_name=core.Fn.import_value("MeshVGWName")
)

self.mesh_crystal_vs= aws_appmesh.VirtualService.from_virtual_service_attributes(
    self,
    "mesh-crystal-vs",
    mesh=self.mesh,
    virtual_service_name=core.Fn.import_value("MeshCrystalVSName")
)

self.mesh_nodejs_vs= aws_appmesh.VirtualService.from_virtual_service_attributes(
    self,
    "mesh-nodejs-vs",
    mesh=self.mesh,
    virtual_service_name=core.Fn.import_value("MeshNodeJsVSName")
)
```

Then we declared the Frontend task definition, already with Proxy configuration needed by the ECS to work with App Mesh:
```python
self.fargate_task_def = aws_ecs.TaskDefinition(
    self, "FrontEndTaskDef",
    compatibility=aws_ecs.Compatibility.EC2_AND_FARGATE,
    cpu='256',
    memory_mib='512',
    proxy_configuration=aws_ecs.AppMeshProxyConfiguration( 
        container_name="envoy",
        properties=aws_ecs.AppMeshProxyConfigurationProps(
            app_ports=[3000],
            proxy_ingress_port=15000,
            proxy_egress_port=15001,
            egress_ignored_i_ps=["169.254.170.2","169.254.169.254"],
            ignored_uid=1337
        )
    )
)
```

![Frontend-Task](../images/frontend-task-def.png)

After that we configured the App Mesh resources. First the Virtual Node, where we reference Crytal and Nodejs virtual services as **_backends_** of Frontend Virtual node:

```python
self.mesh_frontend_vn = aws_appmesh.VirtualNode(
    self,
    "MeshFrontEndNode",
    mesh=self.mesh,
    virtual_node_name="frontend",
    listeners=[aws_appmesh.VirtualNodeListener.http(port=3000)],
    service_discovery=aws_appmesh.ServiceDiscovery.cloud_map(self.fargate_service.cloud_map_service),
    backends=[
        aws_appmesh.Backend.virtual_service(self.mesh_crystal_vs),
        aws_appmesh.Backend.virtual_service(self.mesh_nodejs_vs)
        ]
    
)
```
![Frontend-VN-1](../images/vn-frontend-simple.png)
--
![Frontend-VN-Detail](../images/vn-frontend-detail.png)

Then, we configured the app mesh envoy sidecar `self.envoy_container = self.fargate_task_def.add_container`. Afterwards, we created the Virtual Router for the Frontend. A **virtual router** handles traffic for one or more virtual services within your mesh. After you create a virtual router, you can create and associate routes for your virtual router that direct incoming requests to different virtual nodes. This Virtual router will be used when working with BG/Canary deployments, in a later lecture.

```python
# Creating a App Mesh virtual router
meshVR=aws_appmesh.VirtualRouter(
    self,
    "MeshVirtualRouter",
    mesh=self.mesh,
    listeners=[aws_appmesh.VirtualRouterListener.http(3000)],
    virtual_router_name="FrontEnd"
)

meshVR.add_route(
    "MeshFrontEndVRRoute",
    route_spec=aws_appmesh.RouteSpec.http(
        weighted_targets=[aws_appmesh.WeightedTarget(virtual_node=self.mesh_frontend_vn,weight=1)]
    ),
    route_name="frontend-a"
)
```

![Frontend-VR-Detail](../images/vr-frontend-detail.png)


Once we have configured the virtual router, we proceeded on creating the virtual service, where we used the virtual router as Service provider instead of the Virtual Node as we did in Crystal Virtual Service.

```python
# Asdding mesh virtual service 
self.mesh_frontend_vs = aws_appmesh.VirtualService(self,"mesh-frontend-vs",
    virtual_service_provider=aws_appmesh.VirtualServiceProvider.virtual_router(meshVR),
    virtual_service_name="{}.{}".format(self.fargate_service.cloud_map_service.service_name,self.fargate_service.cloud_map_service.namespace.namespace_name)
)
```

![Frontend-VS-Detail](../images/vs-frontend-router.png)

And lastly, we configure the Virtual Gateway Router. A **gateway route** is attached to a virtual gateway and routes traffic to an existing virtual service. If a route matches a request, it can distribute traffic to a target virtual service. Basically, this a key peace when working with Virtual Gateways, so we can redirect the request to the right services.

```python
# Adding Virtual Gateway Route
self.mesh_gt_router = self.mesh_vgw.add_gateway_route(
    "MeshVGWRouter",
    gateway_route_name="frontend-router",
    route_spec=aws_appmesh.GatewayRouteSpec.http(
        route_target=self.mesh_frontend_vs
    )
)
```
![Frontend-VGR-Detail](../images/vgr-frontend-detail.png)


**Final abstract model of the App architecture after adding the mesh and the constructs to enable App mesh integration**
![final-abstract-mesh-integration](../images/ecs-app-mesh-diagram-Abstract.png)


**Working App:**
![app-running](../images/final-app-working.png)
