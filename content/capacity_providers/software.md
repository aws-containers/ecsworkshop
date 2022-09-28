---
title: "Install and Configure Tools"
chapter: false
weight: 5
---

In the Cloud9 workspace, run the following commands:

## Install and setup prerequisites

```
# Install prerequisite packages
sudo yum -y install jq nodejs python36

# Setting CDK Version
export AWS_CDK_VERSION="1.171.0"

# Install aws-cdk
npm install -g --force aws-cdk@$AWS_CDK_VERSION

# Install cdk packages
pip3 install --user --upgrade aws-cdk.core==$AWS_CDK_VERSION \
aws-cdk.aws_ecs_patterns==$AWS_CDK_VERSION \
aws-cdk.aws_ec2==$AWS_CDK_VERSION \
aws-cdk.aws_ecs==$AWS_CDK_VERSION \
aws-cdk.aws_servicediscovery==$AWS_CDK_VERSION \
aws-cdk.aws_iam==$AWS_CDK_VERSION \
aws-cdk.aws_efs==$AWS_CDK_VERSION \
aws-cdk.aws_autoscaling==$AWS_CDK_VERSION \
aws-cdk.aws_ssm==$AWS_CDK_VERSION \
aws-cdk.aws_appmesh==$AWS_CDK_VERSION \
awscli \
awslogs

# Setting environment variables required to communicate with AWS API's via the cli tools
echo "export AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)" >> ~/.bashrc
echo "export AWS_REGION=\$AWS_DEFAULT_REGION" >> ~/.bashrc
echo "export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)" >> ~/.bashrc
source ~/.bashrc
```
