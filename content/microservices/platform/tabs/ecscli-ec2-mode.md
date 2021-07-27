---
title: "Acceptance and Production"
disableToc: true
hidden: true
---

- Build a VPC, ECS Cluster, and ALB:
 
```
cd ~/environment/ecsdemo-platform

aws cloudformation deploy --stack-name container-demo --template-file cluster-ec2-private-vpc.yml --capabilities CAPABILITY_IAM

aws cloudformation deploy --stack-name container-demo-alb --template-file alb-external.yml
```

