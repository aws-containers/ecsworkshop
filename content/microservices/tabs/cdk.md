---
title: "Embedded tab content"
disableToc: true
hidden: true
---

### Install and setup prerequisites

```bash
# Install prerequisite packages
sudo yum -y install jq nodejs siege

# Install cdk packages
pip3 install --user --upgrade awslogs

#  Verify environment variables required to communicate with AWS API's via the cli tools
grep AWS_DEFAULT_REGION ~/.bashrc 
if [ $? -ne 0 ]; then
    echo "export AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)" >> ~/.bashrc
fi

grep AWS_REGION ~/.bashrc 
if [ $? -ne 0 ]; then
    echo "export AWS_REGION=\$AWS_DEFAULT_REGION" >> ~/.bashrc
fi

grep AWS_ACCOUNT_ID ~/.bashrc 
if [ $? -ne 0 ]; then
    echo "export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)" >> ~/.bashrc
fi 

source ~/.bashrc
```
