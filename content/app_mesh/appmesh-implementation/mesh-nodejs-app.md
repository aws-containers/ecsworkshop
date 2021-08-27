+++
title = "Mesh NodeJs Service"
description = "Mesh NodeJs Service"
weight = 3
+++

At this point, you should have the base platform (VPC, Natgateways, SGs,), the ECS Cluster, Cloud Map and the App Mesh along with the Virtual Gateway.

In this chapter, our goal is to edit your ECS NodeJs App in order to have the Envoy containers running and intercepting the network traffic from your ECS tasks.

**Infrastructure setup:**
![infra-nodejs](../images/ecs-app-mesh-diagram-Infra-NodeJs.png)

### Preparing CDK Code To Deploy App Mesh Resources And ECS Configurations

To enable the nodejs app configuration, please uncomment `lines 65 to 74` on file `~/environment/ecsdemo-nodejs/cdk/app.py` or run this command in the terminal:
```bash
lines=($(grep -Fn '#appmesh-proxy-uncomment' ~/environment/ecsdemo-nodejs/cdk/app.py | cut -f1 -d:))
unstart=$((${lines[0]} + 1))
unend=$((${lines[1]} - 1))
sed -i "${unstart},${unend} s/# //" ~/environment/ecsdemo-nodejs/cdk/app.py 
```

so you can get this result in the file `ecsdemo-nodejs/cdk/app.py`
```python
self.fargate_task_def = aws_ecs.TaskDefinition(
    self, "TaskDef",
    compatibility=aws_ecs.Compatibility.EC2_AND_FARGATE,
    cpu='256',
    memory_mib='512',
    # This will enable App Mesh integration for this Task
    #appmesh-proxy-uncomment
    proxy_configuration=aws_ecs.AppMeshProxyConfiguration( 
        container_name="envoy", #App Mesh side card that will proxy the requests 
        properties=aws_ecs.AppMeshProxyConfigurationProps(
            app_ports=[3000], # nodejs application port
            proxy_ingress_port=15000, # side card default config
            proxy_egress_port=15001, # side card default config
            egress_ignored_i_ps=["169.254.170.2","169.254.169.254"], # side card default config
            ignored_uid=1337 # side card default config
        )
    )
    #appmesh-proxy-uncomment
)
```

The **proxy configuration** will enable the app mesh integration with ECS. for more information regarding the parameter please check out the [official documentation](https://docs.aws.amazon.com/app-mesh/latest/userguide/envoy-config.html) and [CDK Parameters](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_ecs/AppMeshProxyConfiguration.html).


Uncomment `# self.appmesh()` in the line `140`, or execute the following command in the terminal:
```bash
sed -i -e '/self.appmesh()/s/# //' ~/environment/ecsdemo-nodejs/cdk/app.py
```

The `appmesh()` function will add all the required resources into the CF to configure the nodejs app to work with App Mesh. In a moment we will review the resources that were created by this function.

To avoid **[Docker request limits](https://www.docker.com/increase-rate-limits)** we will use our local ECR that we built at the beginning of our configurations. To do so, uncomment line `86` and comment line `85` or use this command:
```bash
#commenting previous line
sed -i -e '/brentley/s/^#*/#/' ~/environment/ecsdemo-nodejs/cdk/app.py
#uncommenting new line
sed -i -e '/amazonaws.com\/ecsdemo-nodejs/s/# //' ~/environment/ecsdemo-nodejs/cdk/app.py
```

so you can get:
```python
# image=aws_ecs.ContainerImage.from_registry("adam9098/ecsdemo-nodejs"),
image=aws_ecs.ContainerImage.from_registry("{}.dkr.ecr.{}.amazonaws.com/ecsdemo-nodejs".format(getenv('AWS_ACCOUNT_ID'), getenv('AWS_DEFAULT_REGION'))),
```

### Deploying Configurations

Install any CDK python prerequisites (libraries) needed by the ecsdemo-nodejs application 
```bash
pip install --upgrade -r ~/environment/ecsdemo-nodejs/cdk/requirements.txt 
```

Confirm that the CDK can synthesize the assembly CloudFormation templates
```bash
cd ~/environment/ecsdemo-nodejs/cdk
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
The information we are going to review moving forward is focused solely on the App Mesh implementations. For explanations regarding Nodejs ECS CDK implementation, please, check out the workshop "[Deploying Microservices to ECS / Nodejs Backend API]({{< ref "microservices/nodejs/" >}})"
{{% /notice %}}

Let’s take a look at what’s being built from the App mesh perspective. 
```python
 def appmesh(self):
        
        # Importing app mesh service
        self.mesh = aws_appmesh.Mesh.from_mesh_arn(
            self,
            "EcsWorkShop-AppMesh",
            mesh_arn=core.Fn.import_value("MeshArn")
        )
        
        # Importing App Mesh virtual gateway
        self.mesh_vgw = aws_appmesh.VirtualGateway.from_virtual_gateway_attributes(
            self,
            "Mesh-VGW",
            mesh=self.mesh,
            virtual_gateway_name=core.Fn.import_value("MeshVGWName")
        )
        
        # App Mesh virtual node configuration
        self.mesh_nodejs_vn = aws_appmesh.VirtualNode(
            self,
            "MeshNodeJsNode",
            mesh=self.mesh,
            virtual_node_name="nodejs",
            listeners=[aws_appmesh.VirtualNodeListener.http(port=3000)],
            service_discovery=aws_appmesh.ServiceDiscovery.cloud_map(self.fargate_service.cloud_map_service),
        )
        
        # App Mesh envoy proxy container configuration
        self.envoy_container = self.fargate_task_def.add_container(
            "NodeJsServiceProxyContdef",
            image=aws_ecs.ContainerImage.from_registry("public.ecr.aws/appmesh/aws-appmesh-envoy:v1.18.3.0-prod"),
            container_name="envoy",
            memory_reservation_mib=170,
            environment={
                "REGION": getenv('AWS_DEFAULT_REGION'),
                "ENVOY_LOG_LEVEL": "debug",
                "ENABLE_ENVOY_STATS_TAGS": "1",
                # "ENABLE_ENVOY_XRAY_TRACING": "1",
                "APPMESH_RESOURCE_ARN": self.mesh_nodejs_vn.virtual_node_arn
            },
            essential=True,
            logging=aws_ecs.LogDriver.aws_logs(
                stream_prefix='/mesh-envoy-container',
                log_group=self.logGroup
            ),
            health_check=aws_ecs.HealthCheck(
                interval=core.Duration.seconds(5),
                timeout=core.Duration.seconds(10),
                retries=10,
                command=["CMD-SHELL","curl -s http://localhost:9901/server_info | grep state | grep -q LIVE"],
            ),
            user="1337"
        )
        
        self.envoy_container.add_ulimits(aws_ecs.Ulimit(
            hard_limit=15000,
            name=aws_ecs.UlimitName.NOFILE,
            soft_limit=15000
            )
        )
        
        # Primary container needs to depend on envoy before it can be reached out
        self.container.add_container_dependencies(aws_ecs.ContainerDependency(
               container=self.envoy_container,
               condition=aws_ecs.ContainerDependencyCondition.HEALTHY
           )
        )
        
        # Enable app mesh Xray observability
        #ammmesh-xray-uncomment
        # self.xray_container = self.fargate_task_def.add_container(
        #     "NodeJsServiceXrayContdef",
        #     image=aws_ecs.ContainerImage.from_registry("amazon/aws-xray-daemon"),
        #     logging=aws_ecs.LogDriver.aws_logs(
        #         stream_prefix='/xray-container',
        #         log_group=self.logGroup
        #     ),
        #     essential=True,
        #     container_name="xray",
        #     memory_reservation_mib=170,
        #     user="1337"
        # )
        
        # self.envoy_container.add_container_dependencies(aws_ecs.ContainerDependency(
        #       container=self.xray_container,
        #       condition=aws_ecs.ContainerDependencyCondition.START
        #   )
        # )
        #ammmesh-xray-uncomment

        self.fargate_task_def.add_to_task_role_policy(
            aws_iam.PolicyStatement(
                actions=['ec2:DescribeSubnets'],
                resources=['*']
            )
        )
        
        self.fargate_service.connections.allow_from_any_ipv4(
            port_range=aws_ec2.Port(protocol=aws_ec2.Protocol.TCP, string_representation="tcp_3000", from_port=3000, to_port=3000),
            description="Allow TCP connections on port 3000"
        )
        
        # Adding policies to work with observability (xray and cloudwath)
        self.fargate_task_def.execution_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("AmazonEC2ContainerRegistryReadOnly"))
        self.fargate_task_def.execution_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("CloudWatchLogsFullAccess"))
        self.fargate_task_def.task_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("CloudWatchFullAccess"))
        #self.fargate_task_def.task_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("AWSXRayDaemonWriteAccess"))
        self.fargate_task_def.task_role.add_managed_policy(aws_iam.ManagedPolicy.from_aws_managed_policy_name("AWSAppMeshEnvoyAccess"))
        
        
        # Adding mesh virtual service 
        self.mesh_nodejs_vs = aws_appmesh.VirtualService(self,"mesh-nodejs-vs",
            virtual_service_provider=aws_appmesh.VirtualServiceProvider.virtual_node(self.mesh_nodejs_vn),
            virtual_service_name="{}.{}".format(self.fargate_service.cloud_map_service.service_name,self.fargate_service.cloud_map_service.namespace.namespace_name)
        )
        
        # Exporting CF (outputs) to make references from other cdk projects.
        core.CfnOutput(self,"MeshNodejsVSARN",value=self.mesh_nodejs_vs.virtual_service_arn,export_name="MeshNodejsVSARN")
        core.CfnOutput(self,"MeshNodeJsVSName",value=self.mesh_nodejs_vs.virtual_service_name,export_name="MeshNodeJsVSName")
              
       
```

When the stack is done building, it will print out all of the outputs for the underlying CloudFormation stack. These outputs are what we use to reference the base platform when deploying the microservices. Below is an example of what the outputs look like:

```bash

Outputs:
ecsworkshop-nodejs.MeshNodeJsVSName = ecsdemo-nodejs.service.local
ecsworkshop-nodejs.MeshNodejsVSARN = arn:aws:appmesh:us-west-2:875448814018:mesh/ecs-mesh/virtualService/ecsdemo-nodejs.service.local

```

### Let's walkthrough The Resources That Were Created

**Virtual Node**
virtual node acts as a logical pointer to a particular task group, such as an Amazon ECS service or a Kubernetes deployment. When you create a virtual node, you must specify a service discovery method for your task group, in our case we used **_AWS Cloud Map_**. Any inbound traffic that your virtual node expects is specified as a listener. Any virtual service that a virtual node sends outbound traffic to is specified as a backend.

![VN-Nodejs-Simple](../images/vn-nodejs-simple.png)

![VN-Nodejs-Detail](../images/vn-nodejs-detail.png)

**Virtual Service**
A virtual service is an abstraction of a real service that is provided by a virtual node directly or indirectly by means of a virtual router. Dependent services call your virtual service by its virtualServiceName, and those requests are routed to the virtual node or virtual router that is specified as the provider for the virtual service. 

There are two important pieces of information to the definition of the virtual service. First, the Virtual Node name that needs to be used as the provider of the virtual service. Second, the service name for the virtual service. The name of a service is a FQDN and is the name used by clients interested in contacting the service. In our example, the Ruby Frontend will issue HTTP requests to `ecsdemo-nodejs.service.local` in order to interact with the NodeJs service.

![VS-NodeJs-Simple](../images/vs-nodejs-simple.png)


**ECS Task Definition Proxy enablement**
It is worth to mention this is a configuration you need to set up within the task definition, so ECS can interact with App Mesh. 

**App Mesh envoy proxy sidecar**
For AppMesh to be able to catch all request and re-direct them, we have to configure sidecar container which we called envoy.

![NodeJs-Task](../images/nodejs-task-def.png)