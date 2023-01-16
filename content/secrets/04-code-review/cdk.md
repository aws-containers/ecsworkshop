---
title: "Embedded tab content"
disableToc: true
hidden: true
---

### Deploy our application, service, and environment

First, let's test the existing code for any errors.

```bash
cd ~/environment/ecsworkshop-secrets-demo
cdk synth
```

This creates the cloudformation templates which output to a local directory `cdk.out`.   Successful output will contain (ignore any warnings generated):

```bash
Successfully synthesized to /home/ec2-user/environment/ecsworkshop-secrets-demo/cdk.out
Supply a stack id (VPCStack, RDSStack, ECSStack) to display its template.
```

(Note this is not a required step as `cdk deploy` will generate the templates again - this is an intermediary step to ensure there are no errors in the stack before proceeding.  If you encounter errors here stop and address them before deployment.)

Then, to deploy this application and all of its stacks, run:

```bash
cdk deploy --all --require-approval never --outputs-file result.json
```

The process takes approximately 10 minutes.  The results of all the actions will be stored in `result.json` for later reference.

{{%expand "Expand to view deployment screenshots" %}}
![CDK Output 1](/images/cdk-output-1.png)
![CDK Output 2](/images/cdk-output-2.png)
{{% /expand%}}

### Code Review

Let's review whats happening behind the scenes.

The repository contains a sample application that deploys a ***ECS Fargate Service***.  The service runs this NodeJS application that connects to a ***AWS RDS Aurora Serverless Database Cluster***.  The credentials for this application are stored in ***AWS Secrets Manager***.

First, let's look at the application context variables:

{{%expand "Review cdk.json" %}}

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/secret-ecs-app.ts",
    "context": {
    "dbName": "tododb",
    "dbUser": "postgres",
    "dbPort": 5432,
    "containerPort": 4000,
    "containerImage": "registry.hub.docker.com/mptaws/secretecs"
    }
}
```

Custom CDK context variables are added to the JSON for the application to consume:

* `dbName` - name of the target database for the tutorial
* `dbUser` - database username
* `dbPort` - database port
* `containerPort` - port on which the container in the ECS cluster runs
* `containerImage` - image name that will be deployed from ECR

These values will be referenced by using the function `tryGetContext(<context-value>)` throughout the rest of the application.
{{% /expand%}}

Next, let's look at the Cloudformation stacks constructs.   The files in `lib` each represent a Cloudformation Stack containing the component parts of the application infrastructure.  

{{%expand "Review lib/vpc-stack.ts" %}}

```ts
import { Construct } from 'constructs';
import { App, Stack, StackProps } from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

export interface VpcProps extends StackProps {
    maxAzs: number;
}

export class VPCStack extends Stack {
    readonly vpc: ec2.Vpc;

    constructor(scope: Construct, id: string, props: VpcProps) {
        super(scope, id, props);

        if (props.maxAzs !== undefined && props.maxAzs <= 1) {
            throw new Error('maxAzs must be at least 2.');
        }

        this.vpc = new ec2.Vpc(this, 'ecsWorkshopVPC', {
            ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),
            subnetConfiguration: [
                {
                    cidrMask: 24,
                    name: 'public',
                    subnetType: ec2.SubnetType.PUBLIC,
                },
                {
                    cidrMask: 24,
                    name: 'private',
                    subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
                },
            ],
        });
    }
}
```

The VPC stack creates a new VPC within the AWS account.   The CIDR address space for this VPC is `10.0.0.0./16`.   It will set up 2 public subnets with NAT Gateways and 2 private subnets with all the appropriate routing information automatically.   An interface is setup to pass in the value for `maxAzs` which is set to 2 in the main application.
{{% /expand%}}

{{%expand "Review lib/rds-stack.ts" %}}

```ts
import { App, StackProps, Stack, Duration, RemovalPolicy, CfnOutput } from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';

export interface RDSStackProps extends StackProps {
    vpc: ec2.Vpc
}

export class RDSStack extends Stack {

    readonly dbSecret: rds.DatabaseSecret;
    readonly postgresRDSserverless: rds.ServerlessCluster;

    constructor(scope: App, id: string, props: RDSStackProps) {
        super(scope, id, props);

        const dbUser = this.node.tryGetContext("dbUser");
        const dbName = this.node.tryGetContext("dbName");
        const dbPort = this.node.tryGetContext("dbPort") || 5432;

        this.dbSecret = new secretsmanager.Secret(this, 'dbCredentialsSecret', {
            secretName: "ecsworkshop/test/todo-app/aurora-pg",
            generateSecretString: {
                secretStringTemplate: JSON.stringify({
                    username: dbUser,
                }),
                excludePunctuation: true,
                includeSpace: false,
                generateStringKey: 'password'
            }
        });

        this.postgresRDSserverless = new rds.ServerlessCluster(this, 'postgresRdsServerless', {
            engine: rds.DatabaseClusterEngine.AURORA_POSTGRESQL,
            parameterGroup: rds.ParameterGroup.fromParameterGroupName(this, 'ParameterGroup', 'default.aurora-postgresql10'),
            vpc: props.vpc,
            enableDataApi: true,
            vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
            credentials: rds.Credentials.fromSecret(this.dbSecret, dbUser),
            scaling: {
                autoPause: Duration.minutes(10), // default is to pause after 5 minutes of idle time
                minCapacity: rds.AuroraCapacityUnit.ACU_8, // default is 2 Aurora capacity units (ACUs)
                maxCapacity: rds.AuroraCapacityUnit.ACU_32, // default is 16 Aurora capacity units (ACUs)
            },
            defaultDatabaseName: dbName,
            deletionProtection: false,
            removalPolicy: RemovalPolicy.DESTROY
        });

        this.postgresRDSserverless.connections.allowFromAnyIpv4(ec2.Port.tcp(dbPort));

        new secretsmanager.SecretRotation(
            this,
            `ecsworkshop/test/todo-app/aurora-pg`,
            {
                secret: this.dbSecret,
                application: secretsmanager.SecretRotationApplication.POSTGRES_ROTATION_SINGLE_USER,
                vpc: props.vpc,
                vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
                target: this.postgresRDSserverless,
                automaticallyAfter: Duration.days(30),
            }
        );

        new CfnOutput(this, 'SecretName', { value: this.dbSecret.secretName });
    }
}
```

Every 30 days, the secret will be rotated and will automatically configure a Lambda function to trigger the rotation using the `single user` method.  More information on the lambdas and methods for credential rotation can be found [here](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html)

{{% /expand%}}

{{%expand "Review lib/ecs-fargate-stack.ts" %}}

Finally, the ECS service stack is defined in `lib/ecs-fargate-stack.ts`

The ECS Fargate cluster application is created here using the `ecs-patterns` library of the CDK.   This automatically creates the service from a given `containerImage` and sets up the code for a load balancer that is connected to the cluster and is public-facing.   The key benefit here is not having to manually add all the boilerplate code to make the application accessible to the world.   CDK simplifies infrastructure creation by abstraction.

The stored credentials created in the RDS Stack are read from Secrets Manager and passed to our container task definition via the `secrets` property.  The secrets unique ARN is passed into this stack as a parameter `dbSecretArn`.

```ts
import { App, Stack, StackProps, CfnOutput } from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecsPatterns from 'aws-cdk-lib/aws-ecs-patterns';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';


export interface ECSStackProps extends StackProps {
  vpc: ec2.Vpc
  dbSecretArn: string
}

export class ECSStack extends Stack {

  constructor(scope: App, id: string, props: ECSStackProps) {
    super(scope, id, props);

    const containerPort = this.node.tryGetContext("containerPort");
    const containerImage = this.node.tryGetContext("containerImage");
    const creds = secretsmanager.Secret.fromSecretCompleteArn(this, 'postgresCreds', props.dbSecretArn);

    const cluster = new ecs.Cluster(this, 'Cluster', {
      vpc: props.vpc,
      clusterName: 'fargateClusterDemo'
    });

    const fargateService = new ecsPatterns.ApplicationLoadBalancedFargateService(this, "fargateService", {
      cluster,
      taskImageOptions: {
        image: ecs.ContainerImage.fromRegistry(containerImage),
        containerPort: containerPort,
        enableLogging: true,
        secrets: {
          POSTGRES_DATA: ecs.Secret.fromSecretsManager(creds)
        }
      },
      desiredCount: 1,
      publicLoadBalancer: true,
      serviceName: 'fargateServiceDemo'
    });

    new CfnOutput(this, 'LoadBalancerDNS', { value: fargateService.loadBalancer.loadBalancerDnsName });
  }

}
```

{{% /expand%}}

{{%expand "Review bin/secret-ecs-app.ts" %}}
Finally, the stacks and the CDK infrastructure application itself are created in `bin/secret-ecs-app.ts`, the entry point for the cdk defined in the `cdk.json` mentioned earlier.

```ts
import { App } from 'aws-cdk-lib';
import { VPCStack } from '../lib/vpc-stack';
import { RDSStack } from '../lib/rds-stack';
import { ECSStack } from '../lib/ecs-fargate-stack';

const app = new App();

const vpcStack = new VPCStack(app, 'VPCStack', {
    maxAzs: 2
});

const rdsStack = new RDSStack(app, 'RDSStack', {
    vpc: vpcStack.vpc,
});

rdsStack.addDependency(vpcStack);

const ecsStack = new ECSStack(app, "ECSStack", {
    vpc: vpcStack.vpc,
    dbSecretArn: rdsStack.dbSecret.secretArn,
});

ecsStack.addDependency(rdsStack);
```

A new CDK app is created `const App = new App()`, and the aforementioned stacks from `lib` are instantiated.  After creating the VPC, the VPC object is passed into the RDS and ECS stacks.  Dependencies are added to ensure the VPC is created before the RDS stack.

When creating the ECS stack, the same VPC object is passed along with a reference to the RDS stack generated `dbSecretArn` so that the ECS stack can look up the appropriate secret.  A dependency is created so that the ECS stack is created after the RDS Stack.
{{% /expand%}}

After deployment finishes, the last step for this tutorial is to get the LoadBalancer URL and run the migration which populates the database.

```bash
url=$(jq -r '.ECSStack.LoadBalancerDNS' result.json)
curl -s $url/migrate | jq
```

(Note that the migration may take a few seconds to connect and run.)

The custom method `migrate` creates the database schema and a single row of data for the sample application. It is part of the sample application in this tutorial.

To view the app, open a browser and go to the Load Balancer URL `ECSST-Farga-xxxxxxxxxx.yyyyy.elb.amazonaws.com` (the URL is clickable in the Cloud9 interface):
![Secrets Todo](/images/secrets-todo.png)

This is a fully functional todo app.  Try creating, editing, and deleting todo items.  Using the information output from deploy along with the secrets stored in Secrets Manager, connect to the Postgres Database using a database client or the `psql` command line tool to browse the database.

As an added benefit of using RDS Aurora Postgres Serverless, you can also use the query editor in the AWS Management Console - find more information **[here](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/query-editor.html)**. All you need is the secret ARN created during stack creation.  Fetch this value at the Cloud9 terminal and copy/paste into the query editor dialog box.   Use the database name `tododb` as the target database to connect.

```bash
aws secretsmanager list-secrets | jq -r '.SecretList[].ARN'
```
