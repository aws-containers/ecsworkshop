---
title: "Acceptance and Production"
disableToc: true
hidden: true
---

#### Navigate to the platform repo and install requirements

```bash
cd ~/environment/ecsdemo-platform/cdk
pip install -r requirements.txt --user
```

#### Confirm that the cdk can synthesize the assembly CloudFormation templates

```bash
cdk synth
```

{{%expand "Fun exercise! Let's count the total number of lines to compare the code written in cdk vs the total lines of generated as CloudFormation. Expand here to see the solution" %}}

```bash
echo -e "Cloudformation Lines==$(cdk synth |wc -l)\nCDK Lines==$(cat app.py|wc -l)"
```

- The end result should look something like this:

```bash
Cloudformation Lines==541
CDK Lines==223
```

{{% /expand %}}

#### View proposed changes to the environment

```bash
cdk diff
```

#### Deploy the changes to the environment

```bash
cdk deploy --require-approval never
```

Let's take a look at what's being built. You may notice that everything defined in the stack is 100% written as python code. We also benefit from the opinionated nature of cdk by letting it build out components based on well architected practices. This also means that we don't have to think about all of the underlying components to create and connect resources (ie, subnets, nat gateways, etc). Once we deploy the cdk code, the cdk will generate the underlying Cloudformation templates and deploy it.

```python
class BaseVPCStack(Stack):

    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # This resource alone will create a private/public subnet in each AZ as well as nat/internet gateway(s)
        self.vpc = ec2.Vpc(
            self, "BaseVPC",
            ip_addresses = ec2.IpAddresses.cidr('10.0.0.0/24'),
        )

        # Creating ECS Cluster in the VPC created above
        self.ecs_cluster = aws_ecs.Cluster(
            self, "ECSCluster",
            vpc=self.vpc,
            cluster_name="container-demo",
            container_insights=True
        )

        # Adding service discovery namespace to cluster
        self.ecs_cluster.add_default_cloud_map_namespace(
            name="service.local",
        )

        # Namespace details as CFN output
        self.namespace_outputs = {
            'ARN': self.ecs_cluster.default_cloud_map_namespace.private_dns_namespace_arn,
            'NAME': self.ecs_cluster.default_cloud_map_namespace.private_dns_namespace_name,
            'ID': self.ecs_cluster.default_cloud_map_namespace.private_dns_namespace_id,
        }

        # Cluster Attributes
        self.cluster_outputs = {
            'NAME': self.ecs_cluster.cluster_name,
            'SECGRPS': str(self.ecs_cluster.connections.security_groups)
        }

        # When enabling EC2, we need the security groups "registered" to the cluster for imports in other service stacks
        if self.ecs_cluster.connections.security_groups:
            self.cluster_outputs['SECGRPS'] = str([x.security_group_id for x in self.ecs_cluster.connections.security_groups][0])
        
        # Frontend service to backend services on 3000
        self.services_3000_sec_group = ec2.SecurityGroup(
            self, "FrontendToBackendSecurityGroup",
            allow_all_outbound=True,
            description="Security group for frontend service to talk to backend services",
            vpc=self.vpc
        )

        # Allow inbound 3000 from ALB to Frontend Service
        self.sec_grp_ingress_self_3000 = ec2.CfnSecurityGroupIngress(
            self, "InboundSecGrp3000",
            ip_protocol='TCP',
            source_security_group_id=self.services_3000_sec_group.security_group_id,
            from_port=3000,
            to_port=3000,
            group_id=self.services_3000_sec_group.security_group_id
        )
        
        # Creating an EC2 bastion host to perform load test on private backend services
        self.amzn_linux = ec2.MachineImage.latest_amazon_linux(
            generation=ec2.AmazonLinuxGeneration.AMAZON_LINUX_2,
            edition=ec2.AmazonLinuxEdition.STANDARD,
            virtualization=ec2.AmazonLinuxVirt.HVM,
            storage=ec2.AmazonLinuxStorage.GENERAL_PURPOSE
        )

        # Instance Role/profile that will be attached to the ec2 instance 
        # Enabling service role so the EC2 service can use ssm
        role = iam.Role(self, "InstanceSSM", assumed_by=iam.ServicePrincipal("ec2.amazonaws.com"))

        # Attaching the SSM policy to the role so we can use SSM to ssh into the ec2 instance
        role.add_managed_policy(iam.ManagedPolicy.from_aws_managed_policy_name("service-role/AmazonEC2RoleforSSM"))

        # Reading user data, to install siege into the ec2 instance.
        with open("stresstool_user_data.sh") as f:
            user_data = f.read()

        # Instance creation
        self.instance = ec2.Instance(
            self, "Instance",
            instance_name="{}-stresstool".format(stack_name),
            instance_type=ec2.InstanceType("t3.medium"),
            machine_image=self.amzn_linux,
            vpc = self.vpc,
            role = role,
            user_data=ec2.UserData.custom(user_data),
            security_group=self.services_3000_sec_group
        )

        # All Outputs required for other stacks to build
        CfnOutput(self, "NSArn", value=self.namespace_outputs['ARN'], export_name="NSARN")
        CfnOutput(self, "NSName", value=self.namespace_outputs['NAME'], export_name="NSNAME")
        CfnOutput(self, "NSId", value=self.namespace_outputs['ID'], export_name="NSID")
        CfnOutput(self, "FE2BESecGrp", value=self.services_3000_sec_group.security_group_id, export_name="SecGrpId")
        CfnOutput(self, "ECSClusterName", value=self.cluster_outputs['NAME'], export_name="ECSClusterName")
        CfnOutput(self, "ECSClusterSecGrp", value=self.cluster_outputs['SECGRPS'], export_name="ECSSecGrpList")
        CfnOutput(self, "ServicesSecGrp", value=self.services_3000_sec_group.security_group_id, export_name="ServicesSecGrp")
        CfnOutput(self, "StressToolEc2Id",value=self.instance.instance_id)
        CfnOutput(self, "StressToolEc2Ip",value=self.instance.instance_private_ip)
```

When the stack is done building, it will print out all of the outputs for the underlying CloudFormation stack. These outputs are what we use to reference the base platform when deploying the microservices. Below is an example of what the outputs look like:

```bash
    ecsworkshop-base

Outputs:
Outputs:
ecsworkshop-base.ECSClusterName = container-demo
ecsworkshop-base.ECSClusterSecGrp = []
ecsworkshop-base.FE2BESecGrp = sg-088f65a73fc5fc48b
ecsworkshop-base.NSArn = arn:aws:servicediscovery:us-east-1:218616270196:namespace/ns-eujlcy2hswslrjcy
ecsworkshop-base.NSId = ns-eujlcy2hswslrjcy
ecsworkshop-base.NSName = service.local
ecsworkshop-base.ServicesSecGrp = sg-088f65a73fc5fc48b
ecsworkshop-base.StressToolEc2Id = i-029e7e175b6754262
ecsworkshop-base.StressToolEc2Ip = 10.0.0.124

Stack ARN:
arn:aws:cloudformation:us-east-1:218616270196:stack/ecsworkshop-base/99b4e7d0-a550-11ed-8358-121ce14ebb21
```

That's it, we have deployed the base platform. Now let's move on to deploying the microservices.
