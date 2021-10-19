---
title: "Install and Configure Tools"
chapter: false
weight: 5
---

## Install and setup prerequisites

Make sure your Cloud9 environment has the proper IAM permissions (proper EC2 instance profile attached and Cloud9 "AWS managed temporary credentials" turned off) as described earlier [here](https://bd4aab6b2cb34b67b37ff409806b25b9.vfs.cloud9.eu-central-1.amazonaws.com/start_the_workshop/workspace/)

```
# Install prerequisite packages
sudo yum -y install jq gettext

# Setting environment variables required to communicate with AWS API's via the cli tools
echo "export AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)" >> ~/.bashrc
echo "export AWS_REGION=\$AWS_DEFAULT_REGION" >> ~/.bashrc
echo "export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)" >> ~/.bashrc
source ~/.bashrc

# Install AWS Session Manager plugin
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo yum install -y session-manager-plugin.rpm
```

jq is a tool that can be used to extract and transform data held in JSON files.

The gettext package includes the envsubst utility, which can be used to substitute the values of environment variables into an input stream.

We will use these tools, along with the Linux utiltity sed, to insert or replace attribute values in various files throughout the workshop. This avoids the need for manual text editing wherever possible.

Note: If you use another Cloud9 OS then Amazon Linux 2 please follow the [instruction](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html#install-plugin-linux) to install AWS session manager plugin.

## Download workshop resources

```
cd ~/environment
git clone https://github.com/aws-containers/ecsworkshop.git
```

## Create the lab environment

```
# create CloudWatch log group
echo "export CW_LOG_GROUP=/aws/ecs/ecs-networking-demo" >> ~/.bashrc
source ~/.bashrc
aws logs create-log-group --log-group-name $CW_LOG_GROUP


# check for ECS service linked role
aws iam get-role --role-name "AWSServiceRoleForECS" || aws iam create-service-linked-role --aws-service-name "ecs.amazonaws.com"

# create ECS Execution role
cat <<EoF > policy_doc.json
{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EoF
echo "export EXEC_ROLE_NAME=ecs-networking-exec-role" >> ~/.bashrc
echo "export EXEC_ROLE_POLICY_ARN=arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy" >> ~/.bashrc
source ~/.bashrc
EXEC_ROLE_ARN=$(aws iam create-role \
        --role-name $EXEC_ROLE_NAME \
        --assume-role-policy-document file://policy_doc.json \
        --query Role.Arn \
        --output text
)
aws iam attach-role-policy \
    --role-name $EXEC_ROLE_NAME \
    --policy-arn $EXEC_ROLE_POLICY_ARN

# create ECS task role
cat <<EoF > task-role-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssmmessages:CreateControlChannel",
                "ssmmessages:CreateDataChannel",
                "ssmmessages:OpenControlChannel",
                "ssmmessages:OpenDataChannel"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:DescribeLogGroups"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:DescribeLogStreams",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:${AWS_REGION}:${AWS_ACCOUNT_ID}:log-group:/aws/ecs/ecs-networking-demo:*"
        }
    ]
}
EoF
echo "export TASK_ROLE_NAME=ecs-networking-task-role" >> ~/.bashrc
echo "export TASK_ROLE_POLICY=ecs-networking-task-role-policy" >> ~/.bashrc
source ~/.bashrc
TASK_ROLE_ARN=$(aws iam create-role \
        --role-name $TASK_ROLE_NAME \
        --assume-role-policy-document file://policy_doc.json \
        --query Role.Arn \
        --output text
)
aws iam put-role-policy \
--role-name $TASK_ROLE_NAME \
--policy-name $TASK_ROLE_POLICY \
--policy-document file://task-role-policy.json

# Create ECS cluster
cd ~/environment/ecsworkshop/content/networking_sd/ecs_networking/setup/
export STACK_NAME=ecs-networking-demo
aws cloudformation create-stack --stack-name $STACK_NAME --template-body file://Cluster-ECS-EC2-2AZ-1NAT.yaml --capabilities CAPABILITY_IAM
# wait a couple of minutes until the following command shows CREATE_COMPLETE
status=$(aws cloudformation describe-stacks --stack-name $STACK_NAME --query 'Stacks[].StackStatus' --output text)
while [ $status != "CREATE_COMPLETE" ]; do echo "CFN status: $status"; sleep 10; status=$(aws cloudformation describe-stacks --stack-name $STACK_NAME --query 'Stacks[].StackStatus' --output text); done

# Create additional env variables from CFN stack output
aws cloudformation describe-stacks --stack-name $STACK_NAME --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' --output text | awk '{ print "export " $1 "=" $2 }' >> ~/.bashrc
source ~/.bashrc
```
