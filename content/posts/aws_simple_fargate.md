---
title: "Running Background Jobs with AWS Fargate, Docker, and Python"
date: 2023-08-27T02:16:00+01:00
tags: 
    - "aws"
    - "docker"
    - "fargate"
    - "python"
    - "terraform"
---
### This guide shows an introduction to ECS and Fargate. The objective is to run a simple python script with dependencies in Fargate. The project creates the necessary infrastructure with Terraform, builds a python docker image, and runs the task in Fargate.

#### Project repo: https://github.com/jcgloria/simple_ecs_fargate

# Prerequisites
- This guide assumes Terraform, Docker, and the AWS CLI are installed and configured.

# Python Container

1. Create a `Dockerfile` for the container you wish to run. In this example we will run a simple python script with dependencies using the official python docker image.
```dockerfile
# Add --platform=linux/amd64 option if running on ARM architecture (e.g. M1 mac)
FROM python:3 

WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD [ "python", "./main.py" ]
```

2. Create a `requirements.txt` file with the dependencies of your python script.
```txt
requests==2.31.0
```

3. Create a `main.py` file with the python script you wish to run.
```python
import requests

print("Hello from Fargate")

r = requests.get('https://jsonplaceholder.typicode.com/todos/1')

print(r.json())
```

# Terraform

With Terraform, we set up all the necessary infrastructure to run the task in Fargate. The infrastructure in this template consists of an ECR repository, an ECS cluster, an ECS task definition, an ECS task execution role, and an ECS task role. 

The task execution role is used by the ECS service to run the task. This role normally provides to ECS access to pull the docker image from ECR and write logs to CloudWatch. On the other hand, the task role is assumed by the task itself and can be used to access other AWS resources. In this terraform template, the policy attached to the task role is defined in the resource `my_ecs_task_role_policy`. As an example, this policy gives full S3 access to the container. Adjust the permissions as needed. 

1. Create a `main.tf` file with the following resources:
```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = var.region
}

# ECR Repository
resource "aws_ecr_repository" "my_repo" {
  name         = var.repo_name
  force_delete = true

  image_scanning_configuration {
    scan_on_push = true
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "cluster" {
  name = var.cluster_name
}

# ECS Task Execution Role
resource "aws_iam_role" "task_execution_role" {
  name = "my_task_execution_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      },
    ] }
  )
}

# ECS Task Execution Policy 
resource "aws_iam_policy" "task_execution_policy" {
  name = "my_ecs_task_execution_policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "logs:CreateLogStream",
          "logs:CreateLogGroup",
          "logs:PutLogEvents"
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}

# Attach the policy to the task execution role
resource "aws_iam_role_policy_attachment" "task_execution_policy_attachment" {
  role       = aws_iam_role.task_execution_role.name
  policy_arn = aws_iam_policy.task_execution_policy.arn
}

# ECS Task Role 
resource "aws_iam_role" "task_role" {
  name = "my_task_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      },
    ] }
  )
}

# ECS Task Policy
resource "aws_iam_policy" "task_role_policy" {
  name = "my_ecs_task_role_policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "s3:*" # Full access to S3, adjust permissions as needed
        ]
        Effect   = "Allow"
        Resource = "*" # You can limit the resources if needed
      }
    ]
  })
}

# Attach the policy to the task role
resource "aws_iam_role_policy_attachment" "task_role_policy_attachment" {
  role       = aws_iam_role.task_role.name
  policy_arn = aws_iam_policy.task_role_policy.arn
}

# ECS Fargate Task Definition + Container Definition
resource "aws_ecs_task_definition" "task" {
  family                   = var.task_name
  requires_compatibilities = ["FARGATE"]
  execution_role_arn       = aws_iam_role.task_execution_role.arn
  task_role_arn            = aws_iam_role.task_role.arn
  network_mode             = "awsvpc"
  cpu                      = 1024
  memory                   = 2048
  container_definitions    = <<DEFINITION
[
  {
    "name": "${var.task_name}",
    "image": "${aws_ecr_repository.my_repo.repository_url}:latest",
    "cpu": 1024,
    "memory": 2048,
    "essential": true,
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "ecs/${var.task_name}",
        "awslogs-region": "${var.region}",
        "awslogs-stream-prefix": "ecs",
        "awslogs-create-group": "true"
      }
    }
  }
]
DEFINITION
}

# Outputs for clearer visibility in the terminal

output "repository_url" {
  value = aws_ecr_repository.my_repo.repository_url
}

output "repository_name" {
  value = aws_ecr_repository.my_repo.name
}

output "cluster_name" {
  value = aws_ecs_cluster.cluster.name
}

output "task_name" {
  value = aws_ecs_task_definition.task.family
}
```

2. Create a `variables.tf` file with the following variables:
```terraform
variable "region" {
  description = "Region to deploy"
  type        = string
  default     = "us-east-1"
}

variable "cluster_name" {
    description = "Name of the ECS cluster"
    type        = string
    default     = "my_cluster"
}

variable "task_name" {
    description = "Name of the ECS task definition"
    type        = string
    default     = "my_task"
}

variable "repo_name" {
    description = "Name of the ECR repository"
    type        = string
    default     = "my_repo"
}
```

3. Initialize the terraform project
```bash
terraform init
```

4. Apply the project in your AWS account
```bash
terraform apply
```

# Docker Image

1. Get AWS credentials to push the docker image to ECR
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <your-repository-url>
```

2. Build the docker image with the ECR repository name
```bash
docker build -t my_repo .
```

3. Tag the docker image with the ECR repository name and the image tag
```bash
docker tag my_repo:latest <your-repository-url>:latest
```

4. Push the docker image to ECR
```bash
docker push <your-repository-url>:latest
```

# Running the task with Fargate

1. Fargate, unlike Lambda, needs to run inside a VPC. Make sure the default VPC is set up in the region the project was deployed. The VPC should have subnets with a route to the internet gateway. 

2. Get the default VPC ID
```bash
DEFAULT_VPC_ID=$(aws ec2 describe-vpcs --query "Vpcs[?IsDefault].VpcId | [0]" --output text)
```

3. Get the default subnets using the default VPC ID
```bash
DEFAULT_SUBNETS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$DEFAULT_VPC_ID" --query "Subnets[*].SubnetId" --output text | tr '\t' ',')
```

4. Run the task
```bash
aws ecs run-task --cluster my_cluster --task-definition my_task --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[$DEFAULT_SUBNETS],assignPublicIp=ENABLED}"
```

# Pull the logs from the task

When the task is done, you can get the latest log stream from the task's log group `ecs/my_task` and print the log events.
```bash
aws logs describe-log-streams --log-group-name ecs/my_task --order-by LastEventTime --descending --limit 1 | jq -r '.logStreams[0].logStreamName' | xargs -I {} aws logs get-log-events --log-group-name ecs/my_task --log-stream-name {}
```