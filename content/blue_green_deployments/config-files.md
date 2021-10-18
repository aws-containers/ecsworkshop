+++
title = "Review the configurations"
chapter = false
weight = 8
+++

#### Open the CodeCommit repository

![navigate-to-codecommit](/images/blue-green-navigate-to-codecommit.gif)

#### CodeBuild uses the `buildspec.yml` for building the container image and pushing to the Elastic Container Registry

* Keep `buildspec.yml` in the root of the source code repository.

{{%expand "Expand to see buildspec.yml" %}}

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - aws ecr get-login-password | docker login --username AWS --password-stdin $REPOSITORY_URI
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Docker build and tagging started on `date`
      - docker build -t $REPOSITORY_URI:latest -t $REPOSITORY_URI:$IMAGE_TAG .
      - echo Docker build and tagging completed on `date`
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Update the REPOSITORY_URI:IMAGE_TAG in task definition...
      - echo Container image to be used $REPOSITORY_URI:$IMAGE_TAG
      - sed -i 's@REPOSITORY_URI@'$REPOSITORY_URI'@g' taskdef.json
      - sed -i 's@IMAGE_TAG@'$IMAGE_TAG'@g' taskdef.json
      - echo update the REGION in task definition...
      - sed -i 's@AWS_REGION@'$AWS_REGION'@g' taskdef.json
      - echo update the roles in task definition...
      - sed -i 's@TASK_EXECUTION_ARN@'$TASK_EXECUTION_ARN'@g' taskdef.json
artifacts:
  files:
    - "appspec.yaml"
    - "taskdef.json"
```

{{% /expand %}}

* The artifacts `appspec.yaml` and `taskdef.json` are used by the CodeDeploy.

* For Amazon ECS compute platform applications, the AppSpec file is used by CodeDeploy to determine your Amazon ECS task definition file. `TASK_DEFINITION` placeholder will be replaced by the CodeDeploy automatically after registering the new `taskdef.json`.

{{%expand "Expand to see appspec.yaml" %}}
```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "<TASK_DEFINITION>"
        LoadBalancerInfo:
          ContainerName: "nginx-sample"
          ContainerPort: 80

```
{{% /expand %}}

* `taskdef.json` is the ECS task definition. New version is created by CodeDeploy for each deployment. We are replacing the below placeholders in the CodeBuild phase of the CodePipeline
    * AWS_REGION
    * REPOSITORY_URI:IMAGE_TAG
    * TASK_EXECUTION_ARN

{{%expand "Expand to see taskdef.json" %}}

```yaml
{
  "containerDefinitions": [
    {
      "name": "nginx-sample",
      "image": "REPOSITORY_URI:IMAGE_TAG",
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "dockerLabels": {
        "name": "nginx-sample"
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/nginx-sample",
          "awslogs-region": "AWS_REGION",
          "awslogs-stream-prefix": "nginx-sample"
        }
      }
    }
  ],
  "taskRoleArn": "TASK_EXECUTION_ARN",
  "executionRoleArn": "TASK_EXECUTION_ARN",
  "family": "nginx-sample",
  "networkMode": "awsvpc",
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "256",
  "memory": "1024"
}

```
{{% /expand %}}

* The placeholders for `appspec.yaml` and `taskdef.json` are being replaced in the `buildspec.yml` 
* The CodeBuild job has the required values available as `ENVIRONMENT` variables in the CDK stack

We have completed the review for the blue/green deployment using CodeDeploy and ECS. Let's cleanup the resources.




