---
title: "Setup Load testing"
chapter: false
weight: 30
---

#### Preparing your Load Test

We now have monitoring enabled for the ECS environment. Let's go ahead and induce manual load to the environment to see how the metrics are shown using Container Insights. To perform the load tests, we will use Siege.

```
# Install Siege for load testing
sudo yum -y install siege
```

Verify Siege is working by typing the below into your terminal window.

```
siege --version
```