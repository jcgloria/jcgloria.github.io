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
1. Create a new terraform project
```bash
terraform init
```

2. Create a `main.tf` file with the following content
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

resource "aws_ecr_repository" "my_repo" {
  name         = var.repo_name
  force_delete = true

  image_scanning_configuration {
    scan_on_push = true
  }
}

resource "aws_ecs_cluster" "cluster" {
  name = var.cluster_name
}


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

resource "aws_iam_policy" "task_execution_policy"{
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
                Effect = "Allow"
                Resource = "*"
            },
        ]
    })
}

resource "aws_iam_role_policy_attachment" "task_execution_policy_attachment" {
    role = aws_iam_role.task_execution_role.name
    policy_arn = aws_iam_policy.task_execution_policy.arn
}


resource "aws_ecs_task_definition" "task" {
  family                   = var.task_name
  requires_compatibilities = ["FARGATE"]
  execution_role_arn       = aws_iam_role.task_execution_role.arn
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

3. Define and adjust the terraform variables file `variables.tf` file with the following content
```terraform
variable "region" {
  description = "Region to deploy"
  type        = string
  default     = "eu-west-2"
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

# Docker

1. Get AWS credentials to push the docker image to ECR
```bash
aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-repository-url>
```

2. Build the docker image with the ECR repository name
```bash
docker build -t <<repo_name>>
```

3. Tag the docker image with the ECR repository name and the image tag
```bash
docker tag <your-image-name>:<your-image-tag> <your-repository-url>:<your-image-tag>
```

4. Push the docker image to ECR
```bash
docker push <your-repository-url>:<your-image-tag>
```

# AWS Fargate

1. Get the default VPC ID of your AWS account
```bash
DEFAULT_VPC_ID=$(aws ec2 describe-vpcs --query "Vpcs[?IsDefault].VpcId | [0]" --output text)
```

2. Get the default subnets of your AWS account
```bash
DEFAULT_SUBNETS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$DEFAULT_VPC_ID" --query "Subnets[*].SubnetId" --output text | tr '\t' ',')
```

3. Run the task
```bash
aws ecs run-task --cluster my_cluster --task-definition my_task --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[$DEFAULT_SUBNETS],assignPublicIp=ENABLED}"
```







