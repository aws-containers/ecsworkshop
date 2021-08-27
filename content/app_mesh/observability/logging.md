+++
title = "Logging"
description = "Logging"
weight = 1
+++

As probably you already noticed, the logging for App Mesh containers was already configured when we deployed our configurations and in this section we will focus on the explanation of the different setups we need to do to have an effective logging.

## Containers Insight
CloudWatch Container Insights is a fully managed service that collects, aggregates, and summarizes Amazon ECS metrics and logs. The CloudWatch Container Insights dashboard gives you access to the following information:

- CPU and memory utilization
- Task and service counts
- Read/write storage
- Network Rx/Tx
- Container instance counts for clusters, services, and tasks

We enabled containers insight when we deployed the based platform at ECS cluster creation, with this configuration in the file `~/environment/ecsdemo-platform/cdk/app.py
```python
self.ecs_cluster = aws_ecs.Cluster(
    self, "ECSCluster",
    vpc=self.vpc,
    cluster_name="container-demo",
    container_insights=True  <--- Here we enabled container insights
)
```

If for some reason you created the cluster without Containers insight enabled, you can enabled it with CLI at a later time using this command:
```bash
# Update ecs cluster settings #
aws ecs update-cluster-settings \
  --cluster $CLUSTER_NAME \
  --settings name=containerInsights,value=enabled
```


**Let's review the Container Insights in the console**

To review the information, you need to go to the Cloudwatch console and select the Insight section, then select Container Insights:
![CloudWatch-Cont-Insights](../images/CloudWatch-Con-Insights.png)

Within this console you will find different views, like the **_Map View_**, that will let you see in a graphical representation the services and task of you ECS cluster and some metrics when you select the service from the map:

![CloudWatch-Map-Detail](../images/CloudWatch-Map-Detail.png)

The other view we have is the **_Performance Monitoring_**, where you will see more metrics related to specific services. This view will let you have a dashboard where you can compare/monitor metrics related to different ECS services.

![CloudWatch-Container-Perf-Insight](../images/CloudWatch-Container-Perf-Insight.png)


For more information related to Containers Insight, follow the instructions provided at [Introducing Container Insights for Amazon ECS](https://aws.amazon.com/blogs/mt/introducing-container-insights-for-amazon-ecs/).

---
#### CLOUDWATCH
When you create your virtual nodes, you have the option to configure Envoy access logs. Once you have enabled it, you will access **CloudWatch Logs** to consume the logs produced by the Envoy proxy. By default, Envoy will produce application and access logs mixed in the same CloudWatch Log file. 

When creating the Virtual Node, you have to indicate the file `/dev/stdout` and to do that, we have configured the App Mesh virtual nodes as follows:

```python
# App Mesh virtual node configuration
self.mesh_crystal_vn = aws_appmesh.VirtualNode(
    self,
    "MeshCrystalNode",
    mesh=self.mesh,
    virtual_node_name="crystal",
    listeners=[aws_appmesh.VirtualNodeListener.http(port=3000)],
    service_discovery=aws_appmesh.ServiceDiscovery.cloud_map(self.fargate_service.cloud_map_service),
    access_log=aws_appmesh.AccessLog.from_file_path("/dev/stdout") <--- Here is where we configured the logging
)
```

The **_ENVOY_LOG_LEVEL_** is configured as parameter when we declare the ENVOY container sidecar within the task. To export only the Envoy access logs (and ignore the other Envoy container logs), you can set the ENVOY_LOG_LEVEL to off. We configured the sidecar as follows:
```python
self.envoy_container = self.fargate_task_def.add_container(
    "CrystalServiceProxyContdef",
    image=aws_ecs.ContainerImage.from_registry("public.ecr.aws/appmesh/aws-appmesh-envoy:v1.18.3.0-prod"),
    container_name="envoy",
    memory_reservation_mib=170,
    environment={
        "REGION": getenv('AWS_DEFAULT_REGION'),
        "ENVOY_LOG_LEVEL": "debug", <---Here is where we configured 
        "ENABLE_ENVOY_STATS_TAGS": "1",
        # "ENABLE_ENVOY_XRAY_TRACING": "1",
        "APPMESH_RESOURCE_ARN": self.mesh_crystal_vn.virtual_node_arn
    },
    .
    .
    .
```

Besides of configuring those sections, we still need to configure the driver that will be used to ingest the logs. In our case we are using Cloudwatch logs driver. If we check the ENVOY container sidecar, you will notice the `logging` configuration:
```
logging=aws_ecs.LogDriver.aws_logs(
    stream_prefix='/mesh-envoy-container',
    log_group=self.logGroup
),
```

and the Cloudwatch logGroup `self.logGroup` was created as follows:
```
self.logGroup = aws_logs.LogGroup(self,"ecsworkshopCrystal",
    # log_group_name="ecsworkshop-crystal",
    retention=aws_logs.RetentionDays.ONE_WEEK
)
```

**Let's check the CloudWatch logs via the console**

There are 2 ways you can get to the Cloudwatch logs. The first one and easiest one, is through out the ECS cluster resources. Firs you have to open the ECS and locate the service you want to check, from there you click on the _Logs_ tab and then select the container you want to see the logs from. This view will show you the aggregated logs of all the task within the service.


![ECS-CW-Service-Aggregated](../images/ECS-CW-Service-Aggregated.png)

If you want to see just the logs related to a specific task/container, you can see that within the task specification from the **_Task_** tab, as follows:

![ECS-CW-Task-Container](../images/ECS-CW-Task-Container.png)


If you want to see the information within CloudWatch logs, you can back to the _**Details**_ tab and then scroll down to the Containers and expand the container you want to see. Then look for the _**Log Configuration section**_ and there you will find the log driver you used. In our case since we used the awslogs driver you will find more information related to the cloudwatch logs and a link to its the dashboard. 
![ECS-CW-Task-Container-Detail](../images/ECS-CW-Task-Container-Detail.png)


The Cloudwatch logs console will allow you to view the logs in **_Logs Insights_**, **_create Log Events, download the logs as file (CSV, ASCII)_** and view the info as plain text.
![CW-Task-Container](../images/CW-Task-Container.png)
