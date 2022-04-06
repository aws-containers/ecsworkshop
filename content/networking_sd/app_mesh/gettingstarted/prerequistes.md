+++
title = "Pre-requistes Config"
description = "Pre-requiste steps for setting up and running workloads in App Mesh"
weight = 2
+++

Before we get started lets create basic configurations for our application

ECS Service Roles
```bash
aws iam get-role --role-name "AWSServiceRoleForECS" || aws iam create-service-linked-role --aws-service-name "ecs.amazonaws.com"
```

Let's clone the application repos so we can deploy the apps to the AWS

```bash
# First we need to clone our repositories locally
cd ~/environment
git clone https://github.com/aws-containers/ecsdemo-platform
git clone https://github.com/aws-containers/ecsdemo-frontend
git clone https://github.com/aws-containers/ecsdemo-nodejs
git clone https://github.com/aws-containers/ecsdemo-crystal
```

Let's pin our CDK libraries to our CLI version

```bash
for i in ecsdemo-*/cdk/requirements.txt ; do sed -i "s/$/==$AWS_CDK_VERSION/g" $i ; done
```
