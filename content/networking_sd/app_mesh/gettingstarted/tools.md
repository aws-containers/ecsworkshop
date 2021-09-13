+++
title = "Pre-requistes software"
description = "Pre-requistes software"
weight = 1
+++


On this workshop we will use [CDK](https://docs.aws.amazon.com/cdk/latest/guide/home.html) to create our environments and deploy the configurations. In the Cloud9 workspace, run the following commands :

```bash
# Install prerequisite packages
sudo yum -y install jq nodejs python36  

# Install ecs cli for local testing
sudo curl -so /usr/local/bin/ecs-cli https://s3.amazonaws.com/amazon-ecs-cli/ecs-cli-linux-amd64-latest
sudo chmod +x /usr/local/bin/ecs-cli

# Setting CDK Version
export AWS_CDK_VERSION="1.116.0"

# Tool to open files on Cloud9 IDE directly from terminar
npm install -g c9

# Install aws-cdk
npm install -g --force aws-cdk@$AWS_CDK_VERSION

# Install cdk packages
pip3 install --user --upgrade awscli awslogs

# Setting environment variables required to communicate with AWS API's via the cli tools
echo "export AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)" >> ~/.bashrc
echo "export AWS_REGION=\$AWS_DEFAULT_REGION" >> ~/.bashrc
echo "export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)" >> ~/.bashrc
source ~/.bashrc
```
