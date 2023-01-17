---
title: "Install and Configure Tools"
chapter: false
weight: 10
---

In the Cloud9 workspace, run the following commands:

## Install and setup prerequisites

```
# Install prerequisite packages
sudo yum -y install jq nodejs python36

# Setting environment variables required to communicate with AWS API's via the cli tools
echo "export AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)" >> ~/.bashrc
echo "export AWS_REGION=\$AWS_DEFAULT_REGION" >> ~/.bashrc
echo "export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)" >> ~/.bashrc
source ~/.bashrc

# Clone the service repo
cd ~/environment
git clone https://github.com/aws-containers/ecsworkshop-efsdemo.git

```
