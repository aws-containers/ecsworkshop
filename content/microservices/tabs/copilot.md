---
title: "Embedded tab content"
disableToc: true
hidden: true
---

### Install and setup prerequisites

```
# Install prerequisites 
sudo yum install -y jq

pip install --user --upgrade awscli

# Install copilot-cli
sudo curl -Lo /usr/local/bin/copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux && sudo chmod +x /usr/local/bin/copilot && copilot --help

# Setting environment variables required to communicate with AWS API's via the cli tools ~ pick ONE of the following:

# IMDSv1
echo "export AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)" >> ~/.bashrc

# IMDSv2
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"` && REGION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region) && echo "AWS_DEFAULT_REGION=$REGION" >> ~/.bashrc

# Reload
source ~/.bashrc

```
