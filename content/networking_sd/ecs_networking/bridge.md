---
title: "Bridge mode"
chapter: false
weight: 30
---

The task utilizes Docker's built-in virtual network which runs inside each Amazon EC2 instance hosting the task. Details can be found [here](https://docs.docker.com/network/bridge/).
The task will get an IP address out of the bridge's network IP range. Containers use this docker0 virtual bridge interface to communicate with endpoints outside of the instance,
using the primary elastic network interface of the instance on which they are running.

In the **task definition** enter "bridge" for [network mode](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#network_mode).

In context to port mapping in bridge network mode, multiple containers can not run on the same host port on the same server (same EC2 instance). For example in order to run two nginx containers with port mapping as 80:80 (host port:container port), one would need two EC2 Instances and this kind of port mapping is called  **Static port mapping**.

There is another way called **Dynamic port mapping** which allows to run multiple containers over the same host using multiple random host ports. In this case port mapping will look like 0:80(host port:container port) and now multiple containers can run on same EC2 Instance, Lets discuss port mapping in more details in following sections:

If you leave the **hostPort** in the container definition empty or '0' dynamic host port mapping will be used. 

```
{
  ...
  "containerDefinitions": [
    ...
    "portMappings": [
      {
          "hostPort": 0,
          "protocol": "tcp",
          "containerPort": 3000
      }
    ],
    ...
  ],
  ...
  "networkMode": "bridge",
  ...
}

```

Note: In the AWS ECS console If you choose \<default\>, ECS will start your container using Docker's default networking mode, which is Bridge on Linux.

Below is a diagram of the bridge mode for EC2 launch type using dynamic port mapping:

![Bridge dynamic mode](/images/ECS_bridge_mode-dynamic.png)

You can define a static port maping as well using **hostPort** like:

```
{
  ...
  "containerDefinitions": [
    ...
    "portMappings": [
      {
          "hostPort": 80,
          "protocol": "tcp",
          "containerPort": 3000
      }
    ],
    ...
  ],
  ...
  "networkMode": "bridge",
  ...
}

```

Below is a diagram of the bridge mode for EC2 launch type using static port mapping:

![Bridge static mode](/images/ECS_bridge_mode-static.png)

When trying to run multiple tasks with static port mapping on the same EC2 instance you will get an error comparable too:

```

Bind for 0.0.0.0:8080 failed: port is already allocated.

```

## Considerations
- Additional network hop and decreased performance by using docker0 virtual bridge interface
- Containers are not addressable with the IP address allocated by Docker
- Host port mapping requires additional operational efforts and considerations
- Single EC2 ENI is shared by multiple containers
- No network policy enforcement for a single container with fine grained security groups possible
- No integration into AWS network observability

For a more detailed comparison between awsvpc and bridge network mode see the following [blog post](https://aws.amazon.com/blogs/compute/introducing-cloud-native-networking-for-ecs-containers/)

## Lab exercise "static port mapping"

One can leverage the new "ECS exec" feature to access containers and check the network configuration.

Note: The executables you want to run in the interactive shell session must be available in the container image!

```
source ~/.bashrc
cd ~/environment/ecsworkshop/content/networking_sd/ecs_networking/setup/
export TASK_FILE=ecs-networking-demo-bridge-mode-stat.json
envsubst < ${TASK_FILE}.template > ${TASK_FILE}
export TASK_DEF=$(aws ecs register-task-definition --cli-input-json file://${TASK_FILE} --query 'taskDefinition.taskDefinitionArn' --output text)
export TASK_ARN=$(aws ecs run-task --cluster ${ClusterName} --task-definition ${TASK_DEF} --enable-execute-command --launch-type EC2 --query 'tasks[0].taskArn' --output text)
aws ecs describe-tasks --cluster ${ClusterName} --task ${TASK_ARN}
# sleep to let the container start
sleep 30
aws ecs execute-command --cluster ${ClusterName} --task ${TASK_ARN} --container nginx --command "/bin/sh" --interactive
```

Inside the container run the following commands:

```
ip a sh
ip link sh
ip r sh
curl localhost:80
# to leave the interactive session type exit
```

Note: One can repeat this with dynamic bridge port mode using TASK_FILE=ecs-networking-demo-bridge-mode-dyn.json

Sample outputs for bridge network mode of a task running a nginx:alpine container using ECS exec:

```
$ aws ecs execute-command  --cluster staging    \
 --task 04812b97f24e460cb9fc70f4282bbb85  --container nginx   --command "/bin/sh"     --interactive

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.

Starting session with SessionId: ecs-execute-command-06ba884b9f06bb3c8
/ # ip a sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:04 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.4/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # ip link sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:04 brd ff:ff:ff:ff:ff:ff
/ # ip r sh
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 scope link  src 172.17.0.4
/ # curl 172.17.0.4:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
```

Another approach is to access the ECS EC2 instance running your task as a priviledged Linux user to observe some details:

```
CONT_INST_ID=$(aws ecs list-container-instances --cluster ${ClusterName} --query 'containerInstanceArns[]' --output text)
EC2_INST_ID=$(aws ecs describe-container-instances --cluster ${ClusterName} --container-instances ${CONT_INST_ID} --query 'containerInstances[0].ec2InstanceId' --output text)
aws ssm start-session --target ${EC2_INST_ID}
```

and to run the following commands inside the instance:

```
sudo -i
docker ps
CONT_ID=$(docker ps --format "{{.ID}} {{.Image}}" | grep nginx:alpine | awk '{print $1}')
HOST_PORT=$(docker inspect $CONT_ID --format "{{.NetworkSettings.Ports}}"| awk '{print $2}' | sed 's/}]]//')
netstat -tulpn | egrep "Program name|$HOST_PORT"
docker network inspect bridge
ip a sh docker0
# to leave the interactive session type exit twice
```

Sample outputs for static host port mapping of a task running a nginx container.

Note the port mapping from "docker ps" which shows that the container port 80/TCP was statically mapped to port 8080 on the ECS host because the task definition file defined "hostPort: 8080". The host process belonging to this port is "docker-proxy"!

```
            "portMappings": [
                {
                    "hostPort": 8080,
                    "protocol": "tcp",
                    "containerPort": 80
                }
```

```
[root@ip-xxx ~]# docker ps
CONTAINER ID        IMAGE                            COMMAND                  CREATED             STATUS                 PORTS                   NAMES
36b4b914e8ba        nginx:alpine                     "/docker-entrypoint.…"   2 minutes ago       Up 2 minutes           0.0.0.0:8080->80/tcp    ecs-ecs-networking-demo-bridge-stat-1-nginx-f0fbbff98de4aeafae01
2585da539bda        amazon/amazon-ecs-agent:latest   "/agent"                 4 hours ago         Up 4 hours (healthy)                           ecs-agent

[root@ip-10-0-101-25 ~]# netstat -tulpn | egrep "Program name|$HOST_PORT"
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp6       0      0 :::8080                 :::*                    LISTEN      25023/docker-proxy 

[root@ip-xxx ~]# docker network inspect bridge
[
    {
        "Name": "bridge",
        ...
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        ...
        "Containers": {
            "f879829d13088138f474be5ab351f6ca54548db656ef6bd7263b95756343c39d": {
                ...
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
    }
]

[root@ip-xxx ~]# ip a sh docker0
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether ...
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0

```
Cleanup the task:

```
aws ecs stop-task --cluster ${ClusterName} --task ${TASK_ARN}
```

## Lab exercise "dynamic port mapping"

One can leverage the new "ECS exec" feature to access containers and check the network configuration.

Note: The executables you want to run in the interactive shell session must be available in the container image!

```
source ~/.bashrc
cd ~/environment/ecsworkshop/content/networking_sd/ecs_networking/setup/
export TASK_FILE=ecs-networking-demo-bridge-mode-dyn.json
envsubst < ${TASK_FILE}.template > ${TASK_FILE}
export TASK_DEF=$(aws ecs register-task-definition --cli-input-json file://${TASK_FILE} --query 'taskDefinition.taskDefinitionArn' --output text)
export TASK_ARN=$(aws ecs run-task --cluster ${ClusterName} --task-definition ${TASK_DEF} --enable-execute-command --launch-type EC2 --query 'tasks[0].taskArn' --output text)
aws ecs describe-tasks --cluster ${ClusterName} --task ${TASK_ARN}
# sleep to let the container start
sleep 30
aws ecs execute-command --cluster ${ClusterName} --task ${TASK_ARN} --container nginx --command "/bin/sh" --interactive
```

Inside the container run the following commands:

```
ip a sh
ip link sh
ip r sh
curl localhost:80
# to leave the interactive session type exit
```

Note: Inside the container the difference between static and dynamic por tmapping is not visible at all!

Sample outputs for bridge network mode of a task running a nginx:alpine container using ECS exec:

```
$ aws ecs execute-command  --cluster staging    \
 --task 04812b97f24e460cb9fc70f4282bbb85  --container nginx   --command "/bin/sh"     --interactive

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.

Starting session with SessionId: ecs-execute-command-06ba884b9f06bb3c8
/ # ip a sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:04 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.4/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # ip link sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:04 brd ff:ff:ff:ff:ff:ff
/ # ip r sh
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 scope link  src 172.17.0.4
/ # curl 172.17.0.4:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
```

Another approach is to access the ECS EC2 instance running your task as a priviledged Linux user to observe some details:

```
CONT_INST_ID=$(aws ecs list-container-instances --cluster ${ClusterName} --query 'containerInstanceArns[]' --output text)
EC2_INST_ID=$(aws ecs describe-container-instances --cluster ${ClusterName} --container-instances ${CONT_INST_ID} --query 'containerInstances[0].ec2InstanceId' --output text)
aws ssm start-session --target ${EC2_INST_ID}
```

and to run the following commands inside the instance:

```
sudo -i
docker ps
CONT_ID=$(docker ps --format "{{.ID}} {{.Image}}" | grep nginx:alpine | awk '{print $1}')
HOST_PORT=$(docker inspect $CONT_ID --format "{{.NetworkSettings.Ports}}"| awk '{print $2}' | sed 's/}]]//')
netstat -tulpn | egrep "Program name|$HOST_PORT"
docker network inspect bridge
ip a sh docker0
# to leave the interactive session type exit twice
```

Sample outputs for dynamic host port mapping of a task running a nginx container.

Note the port mapping from "docker ps" which shows that the container port 80/TCP was dynamically mapped to a random port (in this case 32768) on the ECS host because the task definition file defined "hostPort: 0"! The host process belonging to this port is "docker-proxy"!

```
            "portMappings": [
                {
                    "hostPort": 0,
                    "protocol": "tcp",
                    "containerPort": 80
                }
```

```

[root@ip-xxx ~]# docker ps
CONTAINER ID    IMAGE     COMMAND                  CREATED            STATUS           PORTS                   NAMES
f879829d1308    nginx     "/docker-entrypoint.…"   9 minutes ago      Up 9 minutes     0.0.0.0:32768->80/tcp   ecs-web-bridge-1-web-bridge-aacbafdddd9ad49ef401

[root@ip-10-0-101-25 ~]# docker ps
CONTAINER ID        IMAGE                            COMMAND                  CREATED             STATUS                 PORTS                   NAMES
3de9866ebb5b        nginx:alpine                     "/docker-entrypoint.…"   11 minutes ago      Up 11 minutes          0.0.0.0:32768->80/tcp   ecs-ecs-networking-demo-bridge-dyn-1-nginx-fa80a097a8cd87947600
2585da539bda        amazon/amazon-ecs-agent:latest   "/agent"                 4 hours ago         Up 4 hours (healthy)                           ecs-agent

[root@ip-10-0-101-25 ~]# netstat -tulpn | egrep "Program name|$HOST_PORT"
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp6       0      0 :::32768                :::*                    LISTEN      24284/docker-proxy  

[root@ip-xxx ~]# docker network inspect bridge
[
    {
        "Name": "bridge",
        ...
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        ...
        "Containers": {
            "f879829d13088138f474be5ab351f6ca54548db656ef6bd7263b95756343c39d": {
                ...
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
    }
]

[root@ip-xxx ~]# ip a sh docker0
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether ...
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0

```
Cleanup the task:

```
aws ecs stop-task --cluster ${ClusterName} --task ${TASK_ARN}
```
