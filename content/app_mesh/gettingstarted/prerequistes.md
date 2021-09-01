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

We will create our own repositories in ECR, so we can avoid  Docker Hub requests limits

```bash

# echo '======frontend====='
docker pull adam9098/ecsdemo-frontend && \
aws ecr create-repository --repository-name ecsdemo-frontend --region $AWS_DEFAULT_REGION && \
docker tag adam9098/ecsdemo-frontend $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/ecsdemo-frontend:latest && \
aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com && \
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/ecsdemo-frontend:latest

# echo '======crystal====='
docker pull adam9098/ecsdemo-crystal && \
aws ecr create-repository --repository-name ecsdemo-crystal --region $AWS_DEFAULT_REGION && \
docker tag adam9098/ecsdemo-crystal $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/ecsdemo-crystal:latest && \
aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com && \
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/ecsdemo-crystal:latest


# echo '======nodejs====='
docker pull adam9098/ecsdemo-nodejs && \
aws ecr create-repository --repository-name ecsdemo-nodejs --region $AWS_DEFAULT_REGION && \
docker tag adam9098/ecsdemo-nodejs $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/ecsdemo-nodejs:latest && \
aws ecr get-login-password --region $AWS_DEFAULT_REGION| docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com && \
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/ecsdemo-nodejs:latest

```

Let's clone the application repos so we can the apps to the AWS

```bash
# First we need to clone our repositories locally
cd ~/environment
git clone https://github.com/aws-containers/ecsdemo-platform
git clone https://github.com/aws-containers/ecsdemo-frontend
git clone https://github.com/aws-containers/ecsdemo-nodejs
git clone https://github.com/aws-containers/ecsdemo-crystal
```
