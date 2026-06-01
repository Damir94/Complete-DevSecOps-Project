# Building a Complete DevSecOps Project 

<img width="1000" height="558" alt="1_50Xa4Lct9GaJBhUsuN337g" src="https://github.com/user-attachments/assets/44e1d043-1fae-4462-bc67-00d278b74ec6" />

## Automating AWS Infrastructure with Terraform Cloud & GitHub Actions

### Introduction
This series is my step-by-step journey of building a Netflix-clone application with a full DevSecOps pipeline — everything from infrastructure as code, security scans, and CI/CD automation to monitoring.

And we’re starting right at ground zero — setting up Terraform Cloud for remote state management and using GitHub Actions to automate infrastructure provisioning on AWS.

### Objective
In Part 1, the goal is simple but powerful: build and manage AWS infrastructure using Terraform Cloud — no local state headaches, no manual apply, and no “oops, I deleted my .tfstate file” moments.

We’ll also integrate GitHub Actions to trigger Terraform runs automatically whenever we push code — setting the stage for a fully automated infrastructure pipeline.

By the end of this part, you’ll have:
  - A working Terraform Cloud workspace linked with GitHub
  - Automated infra provisioning via GitHub Actions
  - AWS EC2 instances ready for upcoming Jenkins, monitoring, and Kubernetes setups

### Prerequisites

Before diving in, make sure you’ve got these handy:
  - Basic DevOps knowledge (Git, AWS, Terraform) — no need to be an expert
  - AWS account (Free Tier = fine)
  - Terraform Cloud account (free plan works great)
  - GitHub repo for your Terraform code
  - Optional: Patience, because the first terraform apply always takes longer than you expect

### HandsOn
Directory Structure for the Terraform Code

<img width="720" height="320" alt="1_XqV7MdxvDyVPEOxh9BJ0NA" src="https://github.com/user-attachments/assets/6d5c82ea-c321-40f8-9125-31a6596f63a7" />

### Terraform Scripts for EC2 and other AWS Services(IAM, VPC,etc)

```hcl
# backend.tf 
terraform {
  required_version = "~> 1.13.3"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.14.1"
    }
  }

  cloud {}
}
provider "aws" {
  region = var.aws-region
}

#######################################################################
# gather.tf
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name = "name"
    values = [
      "ubuntu/images/hvm-ssd/ubuntu-noble-24.04-amd64-server-*",
      "ubuntu/images/hvm/ubuntu-noble-24.04-amd64-server-*",
      "ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-*",
      "ubuntu/images/hvm-ebs-gp3/ubuntu-noble-24.04-amd64-server-*"
    ]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"]
}

# iam.tf
resource "aws_iam_role" "role" {
  name = "${local.org}-${local.project}-${local.env}-ssm-iam-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-ssm-iam-role"
    Env  = "${local.env}"
  }
}

resource "aws_iam_role_policy_attachment" "ssm_managed_policy" {
  role       = aws_iam_role.role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "iam-instance-profile" {
  name = "${local.org}-${local.project}-${local.env}-instance-profile"
  role = aws_iam_role.role.name
}

#######################################################################
# vpc.tf
locals {
  org     = "aman"
  project = "netflix-clone"
  env     = var.env
}

resource "aws_vpc" "vpc" {
  cidr_block           = var.cidr-block
  instance_tenancy     = "default"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-vpc"
    Env  = "${local.env}"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-igw"
    env  = var.env
  }

  depends_on = [aws_vpc.vpc]
}

resource "aws_subnet" "public-subnet" {
  count                   = var.pub-subnet-count
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = element(var.pub-cidr-block, count.index)
  availability_zone       = element(var.pub-availability-zone, count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-public-subnet-${count.index + 1}"
    Env  = var.env
  }

  depends_on = [aws_vpc.vpc]
}


resource "aws_route_table" "public-rt" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-public-route-table"
    env  = var.env
  }

  depends_on = [aws_vpc.vpc]
}

resource "aws_route_table_association" "public-rta" {
  count          = 4
  route_table_id = aws_route_table.public-rt.id
  subnet_id      = aws_subnet.public-subnet[count.index].id

  depends_on = [aws_vpc.vpc,
    aws_subnet.public-subnet
  ]
}

resource "aws_security_group" "default-ec2-sg" {
  name        = "${local.org}-${local.project}-${local.env}-sg"
  description = "Default Security Group"

  vpc_id = aws_vpc.vpc.id

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"] // It should be specific IP range
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-sg"
  }
}

#######################################################################
# main.tf
locals {
  instance_names = [
    "jenkins-server",
    "monitoring-server",
    "kubernetes-master-node",
    "kubernetes-worker-node"
  ]
}

resource "aws_instance" "ec2" {
  count                  = var.ec2-instance-count
  ami                    = data.aws_ami.ubuntu.id
  subnet_id              = aws_subnet.public-subnet[count.index].id
  instance_type          = var.ec2_instance_type[count.index]
  iam_instance_profile   = aws_iam_instance_profile.iam-instance-profile.name
  vpc_security_group_ids = [aws_security_group.default-ec2-sg.id]
  root_block_device {
    volume_size = var.ec2_volume_size
    volume_type = var.ec2_volume_type
  }

  tags = {
    Name = "${local.org}-${local.project}-${local.env}-${local.instance_names[count.index]}"
    Env  = "${local.env}"
  }
}

#######################################################################
# variables.tf
variable "aws-region" {}
variable "env" {}
variable "cidr-block" {}
variable "pub-subnet-count" {}
variable "pub-cidr-block" {
  type = list(string)
}
variable "pub-availability-zone" {
  type = list(string)
}
variable "ec2-instance-count" {}
variable "ec2_instance_type" {
  type = list(string)
}
variable "ec2_volume_size" {}
variable "ec2_volume_type" {}

#######################################################################
# dev.auto.tfvars
aws-region            = "us-east-1"
env                   = "dev"
cidr-block            = "10.0.0.0/16"
pub-subnet-count      = 4
pub-cidr-block        = ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20", "10.0.64.0/20"]
pub-availability-zone = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d"]
ec2-instance-count    = 4
ec2_instance_type     = ["t3a.xlarge", "t3a.medium", "t3a.medium", "t3a.medium"]
ec2_volume_size       = 50
ec2_volume_type       = "gp3"
```
