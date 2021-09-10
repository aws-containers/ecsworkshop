+++
title = "Distributed Tracing With Xray"
description = "Distributed Tracing With Xray"
weight = 2
+++

[AWS X-Ray](https://aws.amazon.com/xray/) integrates with AWS App Mesh to manage Envoy proxies for microservices. App Mesh provides a version of Envoy that you can configure to send trace data to the X-Ray daemon running in a container of the same task or pod. 

The AWS X-Ray daemon is a software application that listens for traffic on UDP port 2000, gathers raw segment data, and relays it to the AWS X-Ray API. The daemon works in conjunction with the AWS X-Ray SDKs and must be running so that data sent by the SDKs can reach the X-Ray service.

So we are going to uncomment some configurations on each service, the code is the same on each service so we will just need to explain it once.

First we are going to setup the Xray tracing in the App Mesh Envoy sidecar:
```bash
files=( "ecsdemo-crystal" "ecsdemo-nodejs" "ecsdemo-frontend" "ecsdemo-platform" )
for i in "${files[@]}"
do 
    sed -i -e '/ENABLE_ENVOY_XRAY_TRACING/s/# //' ~/environment/${i}/cdk/app.py
done
```

Now we need to deploy the Xray container sidecar on each service, so the container can send the traces to AWS XRay service and we can ingest the information.

```bash
files=( "ecsdemo-crystal" "ecsdemo-nodejs" "ecsdemo-frontend" "ecsdemo-platform" )
for i in "${files[@]}"
do 
    lines=($(grep -Fn '#ammmesh-xray-uncomment' ~/environment/${i}/cdk/app.py | cut -f1 -d:))
    unstart=$((${lines[0]} + 1))
    unend=$((${lines[1]} - 1))
    sed -i "${unstart},${unend} s/# //" ~/environment/${i}/cdk/app.py 
done
```

And then we will uncomment the permissions that the sidecar needs to send the traces to AWS Xray service:
```bash
files=( "ecsdemo-crystal" "ecsdemo-nodejs" "ecsdemo-frontend" "ecsdemo-platform" )
for i in "${files[@]}"
do 
    sed -i -e '/AWSXRayDaemonWriteAccess/s/# //' ~/environment/${i}/cdk/app.py
done
```

### CDK deployment

Now lest synth each service and deploy them
```bash
files=( "ecsdemo-crystal" "ecsdemo-nodejs" "ecsdemo-frontend" "ecsdemo-platform" )
for i in "${files[@]}"
do 
    cd ~/environment/${i}/cdk
    cdk synth
    cdk diff
    cdk deploy
done
```

### REVIEW TRACES
Once we have deployed them we will need to wait like 5 minutes to start seeing traces. Now access the [X-Ray console](https://console.aws.amazon.com/xray/home). Once in the X-Ray console, select Service Map from the left hand side menu. Wait a few seconds for the service map to render.

Whenever you select a node or edge on an AWS X-Ray service map, the X-Ray console shows a latency distribution histogram. Latency is the amount of time between when a request starts and when it completes. A histogram shows a distribution of latencies. It shows duration on the x-axis, and the percentage of requests that match each duration on the y-axis.

![AWS-X-Ray-Map](../images/AWS-X-Ray-Map.png)

To see the detailed traces for each request in the **_Traces_** tab:
![AWS-X-Ray-Traces](../images/AWS-X-Ray-Traces.png)