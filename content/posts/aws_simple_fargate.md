---
title: "Running background jobs with AWS Fargate, Docker, and Python"
date: 2023-08-27T08:00:00+01:00
draft: true
tags: 
    - "aws"
    - "docker"
    - "fargate"
    - "python"
---

# Terraform

# Docker

Get AWS credentials to push the docker image to ECR
```bash
aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-repository-url>
```
Build the docker image with the ECR repository name
```bash
docker build -t <<repo_name>>
```

Tag the docker image with the ECR repository name and the image tag
```bash
docker tag <your-image-name>:<your-image-tag> <your-repository-url>:<your-image-tag>
```

Push the docker image to ECR
```bash
docker push <your-repository-url>:<your-image-tag>
```

# AWS Fargate

Get the default VPC and subnet IDs
```bash
DEFAULT_VPC_ID=$(aws ec2 describe-vpcs --query "Vpcs[?IsDefault].VpcId | [0]" --output text)
```

```bash
DEFAULT_SUBNETS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$DEFAULT_VPC_ID" --query "Subnets[*].SubnetId" --output text | tr '\t' ',')
```

Run the task
```bash
aws ecs run-task --cluster my_cluster --task-definition my_task --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[$DEFAULT_SUBNETS],assignPublicIp=ENABLED}"
```







