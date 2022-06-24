+++
title = "Build Environment"
chapter = false
weight = 3
+++

#### Install prerequisites

```bash
sudo yum install -y jq
nvm install 16.10.0
nvm use 16.10.0
npm i -g -f aws-cdk@1.128.0
```

#### Navigate to the repo

```bash
git clone https://github.com/aws-containers/ecs-workshop-blue-green-deployments ~/environment/ecs-workshop-blue-green-deployments
cd ~/environment/ecs-workshop-blue-green-deployments
```

#### Build the stack

```bash
npm install
npm run build
npm run test
```

#### Deploy the container image stack 

* This script bootstraps the CDK in the AWS account
* Build a CodeCommit repository and a Codebuild project to create container image

```bash
./bin/scripts/deploy-container-image-stack.sh
```

#### Code review for container image stack
Similar to the previous environments, we follow the same format to build the environment using AWS CDK.

{{%expand "Let's Dive in" %}}

* Create the ECR and CodeCommit repositories

```typescript
// ECR repository for the docker images
this.ecrRepo = new ecr.Repository(this, 'ecrRepo', {
  imageScanOnPush: true
});

// CodeCommit repository for storing the source code
const codeRepo = new codeCommit.Repository(this, 'codeRepo', {
  repositoryName: props.codeRepoName!,
  description: props.codeRepoDesc!
});

```

* Create the CodeBuild project. We provide the ENVIRONMENT variables needed during the build phase. These variables are used for updating the `buildspec.yml`,`taskdef.json` and `appspec.yaml`. We will take a look at these configuration files after the stack is built.

```typescript
// Creating the code build project
this.codeBuildProject = new codeBuild.Project(this, 'codeBuild', {
  role: props.codeBuildRole,
  description: 'Code build project for the application',
  environment: {
    buildImage: codeBuild.LinuxBuildImage.STANDARD_5_0,
    computeType: ComputeType.SMALL,
    privileged: true,
    environmentVariables: {
      REPOSITORY_URI: {
        value: this.ecrRepo.repositoryUri,
        type: BuildEnvironmentVariableType.PLAINTEXT
      },
      TASK_EXECUTION_ARN: {
        value: props.ecsTaskRole!.roleArn,
        type: BuildEnvironmentVariableType.PLAINTEXT
      }
    }
  },
  source: codeBuild.Source.codeCommit({
    repository: codeRepo,
    branchOrRef: 'main'
  })
});
```
{{% /expand %}}

#### Let's push the nginx code sample into CodeCommit repository

* The source code is available [here](https://github.com/aws-containers/ecs-workshop-blue-green-deployments/blob/blue-green-deployment/nginx-sample)
* The [buildspec.yml](https://github.com/aws-containers/ecs-workshop-blue-green-deployments/blob/blue-green-deployment/nginx-sample/buildspec.yml) has placeholders for the variables
* Follow the below steps to upload the code into CodeCommit repository created earlier

```bash
export AWS_DEFAULT_REGION=$(aws configure get region)
export CODE_REPO_NAME=nginx-sample
export CODE_REPO_URL=codecommit::$AWS_DEFAULT_REGION://$CODE_REPO_NAME
cd ~/environment && git clone $CODE_REPO_URL && cd $CODE_REPO_NAME
cp ~/environment/ecs-workshop-blue-green-deployments/nginx-sample/* .
git checkout -b main
git remote -v
git add .
git commit -m "First commit"
git push --set-upstream origin main
```

#### Deploy the pipeline stack

This script executes below steps

* Build the container image for code in CodeCommit using CodeBuild
* Deploy the CodeDeploy and CodePipeline resources for blue/green deployment
* Deploy the AWS Fargate service using the container image

```bash
cd ~/environment/ecs-workshop-blue-green-deployments
./bin/scripts/deploy-pipeline-stack.sh
```
                             
#### Code review of pipeline stack

Similar to the previous environments, we follow the same format to build the environment using AWS CDK.

{{%expand "Let's Dive in" %}}

* Create the task definition
```typescript
// Creating the task definition
const taskDefinition = new ecs.FargateTaskDefinition(this, 'apiTaskDefinition', {
    family: props.apiName,
    cpu: 256,
    memoryLimitMiB: 1024,
    taskRole: props.ecsTaskRole,
    executionRole: props.ecsTaskRole
});
taskDefinition.addContainer('apiContainer', {
    image: ecs.ContainerImage.fromEcrRepository(props.ecrRepository!),
    logging: new ecs.AwsLogDriver({
        logGroup: new log.LogGroup(this, 'apiLogGroup', {
            logGroupName: '/ecs/'.concat(props.apiName!),
            removalPolicy: RemovalPolicy.DESTROY
        }),
        streamPrefix: EcsBlueGreenService.PREFIX
    }),
}).addPortMappings({
    containerPort: props.containerPort!,
    protocol: Protocol.TCP
})
```

* Create load balancers and target groups

```typescript
// Creating an application load balancer, listener and two target groups for Blue/Green deployment
this.alb = new elb.ApplicationLoadBalancer(this, 'alb', {
    vpc: props.vpc!,
    internetFacing: true
});
this.albProdListener = this.alb.addListener('albProdListener', {
    port: 80
});
this.albTestListener = this.alb.addListener('albTestListener', {
    port: 8080
});

this.albProdListener.connections.allowDefaultPortFromAnyIpv4('Allow traffic from everywhere');
this.albTestListener.connections.allowDefaultPortFromAnyIpv4('Allow traffic from everywhere');

// Target group 1
this.blueTargetGroup = new elb.ApplicationTargetGroup(this, 'blueGroup', {
    vpc: props.vpc!,
    protocol: ApplicationProtocol.HTTP,
    port: 80,
    targetType: TargetType.IP,
    healthCheck: {
        path: '/',
        timeout: Duration.seconds(30),
        interval: Duration.seconds(60),
        healthyHttpCodes: '200'
    }
});

// Target group 2
this.greenTargetGroup = new elb.ApplicationTargetGroup(this, 'greenGroup', {
    vpc: props.vpc!,
    protocol: ApplicationProtocol.HTTP,
    port: 80,
    targetType: TargetType.IP,
    healthCheck: {
        path: '/',
        timeout: Duration.seconds(30),
        interval: Duration.seconds(60),
        healthyHttpCodes: '200'
    }
});
```

* Create CloudWatch alarms

```typescript
// CloudWatch Metrics for UnhealthyHost and 5XX errors
const blueUnhealthyHostMetric = EcsServiceAlarms.createUnhealthyHostMetric(props.blueTargetGroup!, props.alb!);
const blue5xxMetric = EcsServiceAlarms.create5xxMetric(props.blueTargetGroup!, props.alb!);
const greenUnhealthyHostMetric = EcsServiceAlarms.createUnhealthyHostMetric(props.greenTargetGroup!, props.alb!);
const green5xxMetric = EcsServiceAlarms.create5xxMetric(props.greenTargetGroup!, props.alb!);

// CloudWatch Alarms for UnhealthyHost and 5XX errors
const blueGroupUnhealthyHostAlarm = this.createAlarm(blueUnhealthyHostMetric, 'blue', 'UnhealthyHost', 2);
const blueGroup5xxAlarm = this.createAlarm(blue5xxMetric, 'blue', '5xx', 1);
const greenGroupUnhealthyHostAlarm = this.createAlarm(greenUnhealthyHostMetric, 'green', 'UnhealthyHost', 2);
const greenGroup5xxAlarm = this.createAlarm(green5xxMetric, 'green', '5xx', 1);
```

* For DeploymentGroup of the CodeDeploy, we have used a custom resource. The CDK construct currently does not support creating a ECS deployment group. First we create the lambda using the `new lambda.Function`, then we create the custom resource using `new CustomResource`

```typescript

// Creating the ecs application
const ecsApplication = new codeDeploy.EcsApplication(this, 'ecsApplication');

// Creating the code deploy service role
const codeDeployServiceRole = new iam.Role(this, 'ecsCodeDeployServiceRole', {
    assumedBy: new ServicePrincipal('codedeploy.amazonaws.com')
});

codeDeployServiceRole.addManagedPolicy(ManagedPolicy.fromAwsManagedPolicyName('AWSCodeDeployRoleForECS'));

// IAM role for custom lambda function
const customLambdaServiceRole = new iam.Role(this, 'codeDeployCustomLambda', {
    assumedBy: new ServicePrincipal('lambda.amazonaws.com')
});

const inlinePolicyForLambda = new iam.PolicyStatement({
    effect: Effect.ALLOW,
    actions: [
        'iam:PassRole',
        'sts:AssumeRole',
        'codedeploy:List*',
        'codedeploy:Get*',
        'codedeploy:UpdateDeploymentGroup',
        'codedeploy:CreateDeploymentGroup',
        'codedeploy:DeleteDeploymentGroup'
    ],
    resources: ['*']
});

customLambdaServiceRole.addManagedPolicy(ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole'))
customLambdaServiceRole.addToPolicy(inlinePolicyForLambda);

// Custom resource to create the deployment group
const createDeploymentGroupLambda = new lambda.Function(this, 'createDeploymentGroupLambda', {
    code: lambda.Code.fromAsset(
        path.join(__dirname, 'custom_resources'),
        {
            exclude: ['**', '!create_deployment_group.py']
        }),
    runtime: lambda.Runtime.PYTHON_3_8,
    handler: 'create_deployment_group.handler',
    role: customLambdaServiceRole,
    description: 'Custom resource to create ECS deployment group',
    memorySize: 128,
    timeout: cdk.Duration.seconds(60)
});

new CustomResource(this, 'customEcsDeploymentGroup', {
    serviceToken: createDeploymentGroupLambda.functionArn,
    properties: {
        ApplicationName: ecsApplication.applicationName,
        DeploymentGroupName: props.deploymentGroupName,
        DeploymentConfigName: props.deploymentConfigName,
        ServiceRoleArn: codeDeployServiceRole.roleArn,
        BlueTargetGroup: props.blueTargetGroupName,
        GreenTargetGroup: props.greenTargetGroupName,
        ProdListenerArn: props.prodListenerArn,
        TestListenerArn: props.testListenerArn,
        TargetGroupAlarms: JSON.stringify(props.targetGroupAlarms),
        EcsClusterName: props.ecsClusterName,
        EcsServiceName: props.ecsServiceName,
        TerminationWaitTime: props.terminationWaitTime
    }
});

this.ecsDeploymentGroup = codeDeploy.EcsDeploymentGroup.fromEcsDeploymentGroupAttributes(this, 'ecsDeploymentGroup', {
    application: ecsApplication,
    deploymentGroupName: props.deploymentGroupName!,
    deploymentConfig: EcsDeploymentConfig.fromEcsDeploymentConfigName(this, 'ecsDeploymentConfig', props.deploymentConfigName!)
});

```

* Code pipeline for the blue/green deployment. This pipeline has three stages - Source, Build and Deploy

```typescript
const pipeline = new codePipeline.Pipeline(this, 'ecsBlueGreen', {
    role: codePipelineRole,
    artifactBucket: artifactsBucket,
    stages: [
        {
            stageName: 'Source',
            actions: [
                new codePipelineActions.CodeCommitSourceAction({
                    actionName: 'Source',
                    repository: codeRepo,
                    output: sourceArtifact,
                    branch: 'main'
                }),
            ]
        },
        {
            stageName: 'Build',
            actions: [
                new codePipelineActions.CodeBuildAction({
                    actionName: 'Build',
                    project: codeBuildProject,
                    input: sourceArtifact,
                    outputs: [buildArtifact]
                })
            ]
        },
        {
            stageName: 'Deploy',
            actions: [
                new codePipelineActions.CodeDeployEcsDeployAction({
                    actionName: 'Deploy',
                    deploymentGroup: ecsBlueGreenDeploymentGroup.ecsDeploymentGroup,
                    appSpecTemplateInput: buildArtifact,
                    taskDefinitionTemplateInput: buildArtifact,
                })
            ]
        }
    ]
});
```

{{% /expand %}}

#### Exporting the Load Balancer URL

```bash
export ALB_DNS=$(aws cloudformation describe-stacks --stack-name BlueGreenPipelineStack --query 'Stacks[*].Outputs[?ExportName==`ecsBlueGreenLBDns`].OutputValue' --output text)
```

* Let's see the deployed version of the application
