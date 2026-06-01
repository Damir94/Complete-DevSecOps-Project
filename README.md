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
### Setting Up Terraform Cloud Terraform State Management
  - Click on the URL to signin/signup for Terraform Cloud: https://app.terraform.io/session
  - Click on Continue with HCP account

<img width="720" height="417" alt="0_GVMb8FRoz7WRMbpx" src="https://github.com/user-attachments/assets/c811ce72-f31e-4b6b-afd8-d30b42819395" />

- We are creating an account on HashiCorp Cloud Platform, as it covers multiple services along with Terraform . This will be useful for your future work. If you want, 
you can also simply sign up for a Terraform account. Now, Click on Sign up

<img width="720" height="377" alt="0_IU6AdRS7PUO2kASb" src="https://github.com/user-attachments/assets/d90fd776-b43e-4387-bcc4-8f5ce3be8720" />

- Now, I will click on continue with GitHub, as I already have a GitHub Account.

<img width="720" height="350" alt="0_zqGgDKYyagSaxUa5" src="https://github.com/user-attachments/assets/45dbb329-c8c0-4e02-a0a0-9c5436822cd9" />

- Once you set up your account, it will look like the snippet below. Although I have already created one organisation.
- Now, we will create our first organisation. So, click on Create organization the " Show " on the top right

<img width="720" height="310" alt="0_9zRuqftZaI8gqU8y" src="https://github.com/user-attachments/assets/c9afd688-8f8f-4df6-a8ce-f93396009e43" />

- To create an organisation, you have to provide three information. Follow the below things:
  - Type of Organisation: If you are using it for a personal use case, go for Personal; else, Business is fine, but you have to pay
  - Organisation Name: It must be a unique name.
  - Email address: Provide your email address, and click on Next.

<img width="720" height="393" alt="0_ewlIQBFvyF73FHku" src="https://github.com/user-attachments/assets/e74e5d5c-1205-4e83-8948-3c7417e8b34a" />

- Once you create the Organization , It will take you to the workspaces section.
- Now, we will create a workspace
- To create a workspace, click on create a workspace

<img width="720" height="388" alt="0_sRHXhScB6xtW681A" src="https://github.com/user-attachments/assets/bc25104d-1c2d-4ab2-8af8-0db9a8f3c84d" />

- After creating the workspace, your workspace will look like the snippet given below

<img width="720" height="346" alt="1_9GbFS7BFWDlswAyR5bFnXw" src="https://github.com/user-attachments/assets/8b71e06a-78dd-4664-81ea-c5ae2cc71905" />

- Now, we have to provide our AWS credentials to Terraform Workspace as our plan, and other Terraform operations will be running on HashiCorp-hosted servers.
- Make sure to create the Environment Variables with the correct Keys and values under the Variables section.
   - AWS_ACCESS_KEY_ID
   - AWS_SECRET_ACCESS_KEY

<img width="720" height="269" alt="1_4OPl_5Sf6bCPFrusQRqP4A" src="https://github.com/user-attachments/assets/f2c61b06-6862-4849-8a9e-6b1f9aff5ee5" />

- Now, we will have to create the token to authenticate with Terraform Cloud from GitHub Actions while running Terraform operations
- Go to your Profile’s Account settings in the top right
- After that, click on Tokens, then click on Create an API token

<img width="720" height="151" alt="1_QKc8Bp0QXlCTe00gDwL9MQ" src="https://github.com/user-attachments/assets/6a0b45a0-212d-415f-bfd1-b066dd2866a0" />

- Provide the description and set the expiration of your token, and click on Generate token

<img width="720" height="302" alt="1_2GPPMj2ks_yxfQ4B2dSwUw" src="https://github.com/user-attachments/assets/cd277e61-fa8b-46c0-b248-b39d8014b454" />

### Automating Infrastructure using Terraform with GitHub Actions
- Now, we will automate our AWS Infrastructure using GitHub Actions
- Below is the workflow script location to add to your Project directory showing below

<img width="720" height="94" alt="1_eMar-GEAA8SuxpexYa_fKg" src="https://github.com/user-attachments/assets/91f88567-ca6b-4113-a3a6-26ff855407c3" />

### The workflow script written below.

```yaml
name: '🔨 Infrastructure Configuring using Terraform on AWS' 

on: 
  workflow_dispatch:
    inputs:
      action:
        type: choice
        description: 'Terraform Action'
        options:
          - plan
          - apply
          - destroy
        required: true
        default: 'plan'

env:
  TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
  TF_WORKSPACE: ${{ secrets.TF_WORKSPACE }}
  TF_CLOUD_ORGANIZATION: ${{ secrets.TF_CLOUD_ORGANIZATION }}
permissions:
  contents: read


jobs:
  infrastructure-setup-using-terraform:
    name: Terraform ${{ github.event.input.actions }}
    runs-on: ubuntu-24.04
    environment: production

    defaults:
      run:
        shell: bash
        working-directory: Terraform

    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Setting Up the Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.13.3"
          terraform_wrapper: true
          cli_config_credentials_token: ${{ secrets. TF_API_TOKEN }}

      - name: Cache Terraform
        uses: actions/cache@v4
        with:
          path: |
            ~/.terraform.d/plugin-cache
            .terraform
          key: ${{ runner.os }}-terraform-${{ hashFiles('**/*.tf') }}
          restore-keys: |
            ${{ runner.os }}-terraform-
          
      - name: Initialising the Terraform
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check --diff

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        if: ${{ github.event.inputs.action == 'plan' }}
        run: terraform plan -input=false

      - name: Terraform Apply
        if: ${{ github.event.inputs.action == 'apply' }}
        run: terraform apply -auto-approve

      - name: Terraform destroy
        if: ${{ github.event.inputs.action == 'destroy' }}
        run: terraform destroy -auto-approve
```
- Now, we need to create three secrets for Terraform Cloud in the GitHub Repo
- Navigate to your Project Repo -> Settings
