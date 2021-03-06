+++
title = "Clean up Compute Resources."
chapter = false
weight = 60
+++

Let's clean up our compute resources first:

{{%expand "Expand here to cleanup the copilot portion of the workshop" %}}

Answer yes when prompted. Make sure the application you delete is called "ecsworkshop"

```bash
cd ~/environment/ecsdemo-frontend/
copilot pipeline delete
```
Once done, run the following:

```bash
copilot app delete 
```
{{% /expand %}}


{{%expand "Expand here to cleanup the cdk portion of the workshop" %}}
## Delete each service stack, and then delete the base platform stack
```bash
cd ~/environment/ecsdemo-frontend/cdk
cdk destroy -f
cd ~/environment/ecsdemo-nodejs/cdk
cdk destroy -f
cd ~/environment/ecsdemo-crystal/cdk
cdk destroy -f
cd ~/environment/ecsdemo-platform/cdk
cdk destroy -f
```
## Clean up log groups associated with services
```bash
python -c "import boto3
c = boto3.client('logs')
services = ['ecsworkshop-frontend', 'ecsworkshop-nodejs', 'ecsworkshop-crystal', 'ecsworkshop-capacityproviders-fargate', 'ecsworkshop-capacityproviders-ec2', 'ecsworkshop-efs-fargate-demo']
for service in services:
    frontend_logs = c.describe_log_groups(logGroupNamePrefix=service)
    print([c.delete_log_group(logGroupName=x['logGroupName']) for x in frontend_logs['logGroups']])"
```
{{% /expand %}}
