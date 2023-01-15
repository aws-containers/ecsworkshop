---
title: "Install and Configure Tools"
chapter: false
weight: 10
---

In the Cloud9 workspace, run the following commands:

## Install and setup prerequisites

```
# Pull down the git repo
cd ~/environment
git clone https://github.com/aws-containers/ecsdemo-migration-to-ecs.git

# Install prerequisite packages
sudo yum -y install jq nodejs python36

# Install SSM session manager plugin
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo yum install -y session-manager-plugin.rpm

# Setting environment variables required to communicate with AWS API's via the cli tools
echo "export AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)" >> ~/.bashrc
echo "export AWS_REGION=\$AWS_DEFAULT_REGION" >> ~/.bashrc
echo "export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)" >> ~/.bashrc
source ~/.bashrc

#Install AWS Copilot cli
curl -Lo copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux && \
chmod +x copilot && \
sudo mv copilot /usr/local/bin/copilot &&\
copilot --help

cat << EOF > ~/.aws/config
[default]
region = ${AWS_DEFAULT_REGION}
output = json
role_arn = $(aws iam get-role --role-name ecsworkshop-admin | jq -r .Role.Arn)
credential_source = Ec2InstanceMetadata
EOF

```
