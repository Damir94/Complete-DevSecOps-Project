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

<img width="1897" height="406" alt="Screenshot 2026-06-12 at 1 25 56 PM" src="https://github.com/user-attachments/assets/c6c90f5f-e90d-4ecc-9f6a-2f4bae76f7b8" />

- To create an organisation, you have to provide three information. Follow the below things:
  - Type of Organisation: If you are using it for a personal use case, go for Personal; else, Business is fine, but you have to pay
  - Organisation Name: It must be a unique name.
  - Email address: Provide your email address, and click on Next.

<img width="1201" height="595" alt="Screenshot 2026-06-12 at 1 28 30 PM" src="https://github.com/user-attachments/assets/9b65125b-0aad-4084-bc6b-9cd8d88393b7" />

- Once you create the Organization , It will take you to the workspaces section.
- Now, we will create a workspace
- To create a workspace, click on create a workspace

<img width="1585" height="606" alt="Screenshot 2026-06-12 at 1 31 49 PM" src="https://github.com/user-attachments/assets/0a429a6e-a65f-446e-ac29-a77a0d65da48" />

- After creating the workspace, your workspace will look like the snippet given below

<img width="1137" height="230" alt="Screenshot 2026-07-21 at 11 19 19 AM" src="https://github.com/user-attachments/assets/fcdc778a-8126-4c3e-84b9-d1ddda6add6b" />

- Now, we have to provide our AWS credentials to Terraform Workspace as our plan, and other Terraform operations will be running on HashiCorp-hosted servers.
- Make sure to create the Environment Variables with the correct Keys and values under the Variables section.
   - AWS_ACCESS_KEY_ID
   - AWS_SECRET_ACCESS_KEY

<img width="1270" height="299" alt="Screenshot 2026-07-21 at 11 20 56 AM" src="https://github.com/user-attachments/assets/1469b6e7-9f67-4756-a85a-62ebe358fb65" />

- Now, we will have to create the token to authenticate with Terraform Cloud from GitHub Actions while running Terraform operations
- Go to your Profile’s Account settings in the top right
- After that, click on Tokens, then click on Create an API token

<img width="720" height="151" alt="1_QKc8Bp0QXlCTe00gDwL9MQ" src="https://github.com/user-attachments/assets/6a0b45a0-212d-415f-bfd1-b066dd2866a0" />

- Provide the description and set the expiration of your token, and click on Generate token

<img width="720" height="302" alt="1_2GPPMj2ks_yxfQ4B2dSwUw" src="https://github.com/user-attachments/assets/cd277e61-fa8b-46c0-b248-b39d8014b454" />

### Automating Infrastructure using Terraform with GitHub Actions
- Now, we will automate our AWS Infrastructure using GitHub Actions
- Below is the workflow script location to add to your Project directory showing below

<img width="882" height="189" alt="Screenshot 2026-07-21 at 11 23 24 AM" src="https://github.com/user-attachments/assets/5bc20c52-b80e-4ca4-9696-8cade1d24d1e" />

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

<img width="1567" height="125" alt="Screenshot 2026-07-21 at 11 25 59 AM" src="https://github.com/user-attachments/assets/87de9767-d83b-4619-98d4-3f1b0f252fa7" />

- Expand the Secrets and variables, then click on Actions

<img width="420" height="291" alt="Screenshot 2026-07-21 at 11 28 06 AM" src="https://github.com/user-attachments/assets/6b05b8aa-e62f-4d5e-a40a-bcd5773682ee" />

- Then, add the three variables with correct values
  - TF_API_TOKEN
  - TF_CLOUD_ORGANIZATION
  - TF_WORKSPACE

<img width="1544" height="753" alt="Screenshot 2026-07-21 at 11 29 09 AM" src="https://github.com/user-attachments/assets/e2a8fef9-96ad-4c79-b7d1-e4a3f05bf064" />

### Now, we are ready to deploy our Infrastructure using Terraform, Terraform Cloud, and GitHub Actions on AWS
- Go to the Actions of the GitHub Repository
- Select the apply from the drop-down

<img width="1913" height="783" alt="Screenshot 2026-07-21 at 11 33 25 AM" src="https://github.com/user-attachments/assets/8326e736-f2c8-4c65-a42d-5d17a89ddf7c" />

- Apply is successful

<img width="1539" height="919" alt="Screenshot 2026-07-21 at 11 35 15 AM" src="https://github.com/user-attachments/assets/c0745118-9834-4b78-b075-aaf539012df6" />

- You can validate the resource from Terraform Cloud as well

<img width="1873" height="873" alt="Screenshot 2026-07-21 at 11 36 17 AM" src="https://github.com/user-attachments/assets/de55c34a-33cf-4187-9435-239ffc422c37" />

- Now, let’s go to the AWS Console and see whether our EC2s are there or not

<img width="1461" height="320" alt="Screenshot 2026-07-21 at 11 37 52 AM" src="https://github.com/user-attachments/assets/ca2b3fc7-13a7-4afb-a710-41dba944e0f3" />

- You’ve just automated your AWS infrastructure with Terraform Cloud and GitHub Actions — the clean, modern way.

##  Setting Up Jenkins, Docker, SonarQube, and Trivy for CI/CD

### Introduction
- In this part, we’ll turn a plain EC2 instance into a fully functional CI/CD control tower, capable of building, testing, and scanning code automatically. Think of it as teaching your pipeline to not just deploy, but deploy securely.

### Objective
- Set up a secure CI/CD environment that includes:
  - Jenkins — the orchestrator
  - Docker — container runtime for isolated builds
  - SonarQube — code quality and static analysis
  - Trivy — vulnerability scanner for images and dependencies
  - kubectl — for upcoming Kubernetes integrations
  - Slack notifications — so you know when things break (and when they don’t)
- By the end, you’ll have a Jenkins-driven build pipeline ready to integrate into the larger DevSecOps flow.

### Hands-On
- As all four machines are running. We are going to start with the Jenkins Server to configure.
- Select the Jenkins Server and connect with Session Manager

<img width="1622" height="238" alt="Screenshot 2026-07-21 at 11 39 54 AM" src="https://github.com/user-attachments/assets/b170dfcb-b355-42a8-9bf4-995dc2194a01" />

- Log in as the ubuntu user
```bash
sudo su ubuntu
cd
```
### Jenkins Installation Guide (Ubuntu)

Step 1: Update your system

```bash
sudo apt update && sudo apt upgrade -y
```

<img width="921" height="311" alt="Screenshot 2026-07-21 at 11 42 58 AM" src="https://github.com/user-attachments/assets/62d6cf6e-621d-4e3f-9003-826e67c55cd3" />

Step 2: Install Java (Jenkins needs Java) Jenkins works best with OpenJDK 21 now.

```bash
sudo apt install fontconfig openjdk-21-jre
```
```bash
java -version
```

<img width="904" height="91" alt="Screenshot 2026-07-21 at 12 02 38 PM" src="https://github.com/user-attachments/assets/82251ce8-6199-47e9-b28c-8948eace39f2" />

Step 3: Add Jenkins official repository & key

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Step 4: Install Jenkins

```bash
sudo apt update
sudo apt install jenkins
```

Step 5: Start & enable Jenkins

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

- Check status:
```bash
sudo systemctl status jenkins
```

<img width="1564" height="295" alt="Screenshot 2026-07-21 at 12 04 43 PM" src="https://github.com/user-attachments/assets/045b08fe-9760-477c-b688-5e7dc04be309" />

- Let’s validate whether Jenkins is installed or not by accessing it.
- Get the Public IP of your Jenkins Server and add the 8080 Port

<img width="720" height="417" alt="1_Rg0PtJLSZrTb4_xBG_ji1A" src="https://github.com/user-attachments/assets/328731d5-ed1d-4da5-9254-b40a71c13f13" />

- To get the password, go to the machine and run the command below
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

<img width="720" height="57" alt="1_3yivBdY9QLINg0X3GVZ8Wg" src="https://github.com/user-attachments/assets/246cbdc7-9120-461a-a023-0dfa60a20bc8" />

- Click on Install suggested plugins to download the plugins

<img width="720" height="418" alt="1_igZxCGqn8TLyx5zUbY5e2Q" src="https://github.com/user-attachments/assets/ed0f961e-4d2a-4fad-8f4e-b8822c0b4845" />

- Now, we will create one user instead of using Admin
- Provide the required information

<img width="795" height="510" alt="Screenshot 2026-07-21 at 12 10 08 PM" src="https://github.com/user-attachments/assets/4b743bf0-43d1-42c9-a0c6-0bef265d97f6" />

- Click on Save and Finish

<img width="964" height="354" alt="Screenshot 2026-07-21 at 12 10 53 PM" src="https://github.com/user-attachments/assets/fd8c5e5f-ae90-4ee5-b392-99407ea74494" />

- We are ready with our Jenkins Server

<img width="1456" height="630" alt="Screenshot 2026-07-21 at 12 11 39 PM" src="https://github.com/user-attachments/assets/5434afd5-76d3-4c81-ab83-f4c2e72451d2" />

- Now, we have to install multiple tools on Jenkins, in which Docker is first.
- Run the command below to install docker
```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart docker
sudo chmod 777 /var/run/docker.sock
```

<img width="1068" height="275" alt="Screenshot 2026-07-21 at 12 14 31 PM" src="https://github.com/user-attachments/assets/9a001ec4-6587-4091-9610-a1058da7aa4a" />

- After installing Docker, we will be going to install Sonarqube for Code Quality.
- We have two options to install Sonarqube which are installing Sonarqube on EC2 directly or installing Sonarqubeusing Docker container
- We will be using docker to install it Sonarqube
```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:community
```

<img width="1660" height="475" alt="Screenshot 2026-07-21 at 12 16 04 PM" src="https://github.com/user-attachments/assets/1c1b44d5-1d9a-4282-af1e-e7af3b2e0c50" />

- The Sonarqube docker container is running, and now we will access it using the Same Jenkins IP with Port 9000
- To access SonarQube, the username and password are admin

<img width="1726" height="899" alt="Screenshot 2026-07-21 at 12 17 44 PM" src="https://github.com/user-attachments/assets/efa0ce90-af61-4003-b5d4-cfff07cb97fb" />

- Update the password of your SonarQube

<img width="720" height="244" alt="1_YHOpkOpdzxxGENNO2sa8Ng" src="https://github.com/user-attachments/assets/c2891141-0aed-4c9d-a4f9-c42300e99dab" />

- Here is the SonarQube dashboard

<img width="1739" height="703" alt="Screenshot 2026-07-21 at 12 20 04 PM" src="https://github.com/user-attachments/assets/6bac1c17-82e2-42e1-acf0-493a653828f1" />

- Now, we have to install our next tool, which is trivy
```bash
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
trivy --version
```

<img width="846" height="288" alt="Screenshot 2026-07-21 at 12 21 42 PM" src="https://github.com/user-attachments/assets/5c6f4873-09b6-4984-b1a5-7d6096559916" />

- Now, we will install kubectl utility
```bash
curl -LO https://dl.k8s.io/release/v1.33.5/bin/linux/amd64/kubectl
curl -LO https://dl.k8s.io/release/v1.33.5/bin/linux/amd64/kubectl.sha256
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

<img width="1034" height="354" alt="Screenshot 2026-07-21 at 12 22 46 PM" src="https://github.com/user-attachments/assets/1aee0386-4e2b-494f-91ab-b4a0d4fa367e" />

- Now, we will be setting up Slack for notification of our Pipelines whenever they finish with any status like Failed, Succeeded, or Aborted.
- For that, we have to install the Jenkins plugin called Slack Notification

<img width="720" height="144" alt="1_dUNgAOmL_iuZnCGNGZS3MQ" src="https://github.com/user-attachments/assets/9d28bca3-df41-4c91-ab9e-5e1ba34eb015" />

- Plugin is installed. Check the Restart Jenkins options

<img width="720" height="343" alt="1_FHeIQKki8x9EG9EL9trGeQ" src="https://github.com/user-attachments/assets/76ee88fb-6d82-4d5d-917e-1954b036f8ba" />

- Now, we have to configure the authentication from the Slack side
- Go to the Link: https://api.slack.com/apps and make sure you have already signed in with your official Slack account
- Click on Create New App

<img width="1025" height="407" alt="Screenshot 2026-07-21 at 12 31 41 PM" src="https://github.com/user-attachments/assets/e28bb571-88cd-4d88-9381-d3f105d5c28b" />

- We will be creating an app using a manifest. So, click on it

<img width="1017" height="554" alt="Screenshot 2026-07-21 at 12 32 56 PM" src="https://github.com/user-attachments/assets/ed99dd0c-0025-4a2e-ba0f-1d553ba58cf1" />

- Select the correct workspace for Slack

<img width="1010" height="572" alt="Screenshot 2026-07-21 at 12 33 34 PM" src="https://github.com/user-attachments/assets/e64f9204-5e01-43d4-8b6c-f75158b55a0c" />

- We will be writing the manifest in YAML format. So, click on YAML and remove the existing/default manifest and replace it with the below manifest, and click on Next
```yaml
display_information:
  name: Jenkins
features:
  bot_user:
    display_name: Jenkins
    always_online: true
oauth_config:
  scopes:
    bot:
      - channels:read
      - chat:write
      - chat:write.customize
      - files:write
      - reactions:write
      - users:read
      - users:read.email
      - groups:read
settings:
  org_deploy_enabled: false
  socket_mode_enabled: false
  token_rotation_enabled: false
```

<img width="951" height="734" alt="Screenshot 2026-07-21 at 12 34 35 PM" src="https://github.com/user-attachments/assets/04f7d2e4-e08e-4ec8-a0f1-d79ef1ee91ea" />

- Review the provided information and click on Create

<img width="1024" height="558" alt="Screenshot 2026-07-21 at 12 35 20 PM" src="https://github.com/user-attachments/assets/9e9ff12e-c1cd-45d5-b60d-3ec8f1396407" />

- After creating the app, your app will look like this

<img width="1031" height="616" alt="Screenshot 2026-07-21 at 12 36 12 PM" src="https://github.com/user-attachments/assets/8fcd3aff-8ec3-4bee-b4ee-2c1c4079aff4" />

- You can configure the Display information

<img width="592" height="588" alt="Screenshot 2026-07-21 at 12 39 39 PM" src="https://github.com/user-attachments/assets/b8ce1045-9978-428e-adec-ba956d8884bb" />

- Now, install the App on our Slack workspace
- Click on Install

<img width="918" height="340" alt="Screenshot 2026-07-21 at 12 41 23 PM" src="https://github.com/user-attachments/assets/bca97789-ebd6-41f0-9a98-b61535832a4c" />

- It will prompt you to reconfirm a few things. After confirming, click on Allow

<img width="1014" height="591" alt="Screenshot 2026-07-21 at 12 42 13 PM" src="https://github.com/user-attachments/assets/13be2cf7-736b-429e-8023-33342f222483" />

- You will get the Bot User Token

<img width="907" height="380" alt="Screenshot 2026-07-21 at 12 43 06 PM" src="https://github.com/user-attachments/assets/ae2850b9-4e80-46cb-8174-dfdf63b4bc62" />

- Also, if you check your Slack. You will see one app added called Jenkins

<img width="947" height="222" alt="Screenshot 2026-07-21 at 12 44 16 PM" src="https://github.com/user-attachments/assets/f3fab73a-adc9-468f-a51d-31c28120c6bc" />

- Now, we have to copy the Bot User Token and use it in Jenkins
- Go to the Jenkins -> Manage Jenkins -> System and look for Slack

<img width="1641" height="483" alt="Screenshot 2026-07-21 at 12 45 59 PM" src="https://github.com/user-attachments/assets/9694a6b0-ab35-4ebb-988f-037570e80648" />

- We have to add a secret by clicking on +Add, which we copied in the earlier steps
- Click on Add

<img width="688" height="558" alt="Screenshot 2026-07-21 at 12 48 08 PM" src="https://github.com/user-attachments/assets/f1fa7838-ef62-49ab-b560-eab8ae329a3f" />

- After adding the secret, we have to invite the Jenkins app that we created in our Slack channel, as my channel is private. But if your channel is public, you don’t need to invite
```bash
/invite @Jenkins
```

<img width="957" height="407" alt="Screenshot 2026-07-21 at 12 51 35 PM" src="https://github.com/user-attachments/assets/67bf06f4-281c-4184-af24-9096e79f2788" />

- Now, add the workspace name and channel name of your Slack where you want to get the notification of your Jenkins Pipeline and test the connection. As you can see, our Test Connection is successful.

<img width="720" height="290" alt="1_Ac4SK7K1bfsAK35J-vn2nw" src="https://github.com/user-attachments/assets/45577b0d-3055-4a9e-9659-d4da439bd8d5" />

- You can also validate from here your Slack channel as it sends you a ping

<img width="701" height="139" alt="Screenshot 2026-07-21 at 12 56 27 PM" src="https://github.com/user-attachments/assets/b8c415fe-6b3e-4ec5-842a-4022463ae22b" />

- Once Slack is working fine, then click on Save and Apply

<img width="1539" height="628" alt="Screenshot 2026-07-21 at 12 57 37 PM" src="https://github.com/user-attachments/assets/f0046264-ca0a-4e49-8521-867780141153" />

### Conclusion
- With this setup, your DevOps engine is now fully functional and security-aware.
- You’ve got Jenkins running with Docker, SonarQube, and Trivy — ready to handle real workloads and enforce quality gates automatically.

## Creating a Secure Jenkins Pipeline for Code Scanning and Docker Image Builds

- Now in Part 3, it’s time to make Jenkins do some real work.
- We’re building our first end-to-end pipeline — one that doesn’t just build and scan but also checks your dependencies, analyses your code, and even packages everything neatly into Docker images.

### Objective
- By the end of this part, you’ll have:
  - A fully configured Jenkins pipeline that runs through multiple DevSecOps stages.
  - Integrations for SonarQube, Trivy, OWASP Dependency Check, and Docker.
  - A functional TMDB API key setup for your Netflix Clone microservice
  - A robust pipeline flow — from checkout → code scan → dependency scan → Docker build → image scan → push to registry.

### Hands-On
- Now, we will install two Jenkins plugins:
  - SonarQube Scanner - To Integrate with Sonar
  - NodeJS - To build our code as it is NodeJS based application 

<img width="720" height="204" alt="1__-jHixw0nElMTz5Oslmarw" src="https://github.com/user-attachments/assets/f41c0534-4b91-4877-872b-7c76d2110b00" />

- Now, Restart your Jenkins

<img width="720" height="262" alt="1_GDjKoWkV0d-KejPGoMfVpg" src="https://github.com/user-attachments/assets/2e614f58-0c0b-49c3-bb80-494f35262613" />

- It’s time to configure the installed plugins
- Go to the Tools section under Manage Plugins
- Configure NodeJS by providing the desired name and using the compatible version, and click on Save and Apply

<img width="720" height="391" alt="1_7Ba21WezSBCWkk3JoCdQag" src="https://github.com/user-attachments/assets/56e8f6d6-e8ff-4592-bf6f-91d7bd870a12" />

- It’s time to configure SonarQube. So go to the Sonar dashboard. Click on User

<img width="720" height="204" alt="1_snyhDAY5jp50oHhQ9T_vlw" src="https://github.com/user-attachments/assets/5369b882-3c51-45aa-b6ad-9aeea6b7b9cd" />

- Click on the three dots showing under the Tokens

<img width="720" height="161" alt="1_BLvpBl40_xHP_mYnQ1GA0Q" src="https://github.com/user-attachments/assets/27cd9adc-0006-4abf-8cf6-c794a67caaae" />

- Provide the name of your token and copy the token

<img width="720" height="345" alt="1_JfY5fsx0_h-P79hdOQ5T8Q" src="https://github.com/user-attachments/assets/52581780-3f17-44d9-bef5-90b4dba067db" />

- Now, we have to add the copied token to Jenkins Secrets
- Click on Add credentials

<img width="720" height="167" alt="1_NOHSuVvLU9CT8yUT9wQpBg" src="https://github.com/user-attachments/assets/3ef1d5be-1ca4-4dc0-ac76-6ae3ee333916" />

- Provide the correct secrets and other information, and click on Create

<img width="720" height="329" alt="1_ZVZSHc9S1tXpkxmkzYIiYA" src="https://github.com/user-attachments/assets/cc96cddb-9ba4-4def-a8a9-ce8a32571bb7" />

- Go to Manage Jenkins -> System
- Click on Add SonarQube

<img width="720" height="181" alt="1_aUothwzWqcwGw1gEJq8McA" src="https://github.com/user-attachments/assets/20c6b76a-c88c-4bac-858a-97ef9f927b0e" />

- Please provide the name sonar-server with the Server URL and select the credentials that we have added.

<img width="720" height="310" alt="1_K6lu9MKNo7acJ2Tv35CTgQ" src="https://github.com/user-attachments/assets/750346e6-b39f-4e55-8bb5-ed5bbfa45556" />

- Now, we have to configure Sonar under the Tools section of Manage Plugins
- Go to Manage Jenkins -> Tools

<img width="720" height="154" alt="1_K3AhkPjhglREcHd7mNULzg" src="https://github.com/user-attachments/assets/13fd5bfb-d807-468a-9a98-6e3975ebacc7" />

- Provide the name sonar-server and select the latest version of SonarQube.

<img width="720" height="281" alt="1_qdI3c0GO-W6GcXu7p0MhoQ" src="https://github.com/user-attachments/assets/bb0c4cc4-8eea-40e3-b471-5422988adccb" />

- Now, we need to create a webhook for Quality Gates. Click on Configuration and select Webhooks.

<img width="720" height="154" alt="1_ZN50y5OJkRn6JbIEzrm4Lg" src="https://github.com/user-attachments/assets/87bffd37-6e59-49eb-a548-5c626af93df6" />

- Click on Create

<img width="720" height="183" alt="1_y4JNp3DDvBOSt0RS6hwv1w" src="https://github.com/user-attachments/assets/5a8cd294-ec67-4dab-8b24-ae340f3fe661" />

- Provide the correct information for your Jenkins Server and click on Create

<img width="720" height="343" alt="1_Q7KllgiPbS8fwG7y4-9PFQ" src="https://github.com/user-attachments/assets/bcc5bcaa-7879-4028-8956-6999f8442a79" />

- The Webhook will be showing the below snippet

<img width="720" height="210" alt="1_Lh7FrIrH7cu8seU7esOFZA" src="https://github.com/user-attachments/assets/86d65b15-7f3e-4e0c-94d0-7d5029cf8efc" />

- Now, we will be creating the Project on SonarQube
- Click on Create a local project

<img width="720" height="307" alt="1_zHhLK5L-59Q5RXcVfDnFIw" src="https://github.com/user-attachments/assets/47177cff-cafc-4a28-8f9c-4829140e6c3b" />

- Provide the correct information and click on Create

<img width="720" height="261" alt="1_YV3B3YcNIoCronyJL31umQ" src="https://github.com/user-attachments/assets/910fc8a6-c214-43d0-abe8-91a11c82e8d9" />

- Now, click on locally

<img width="720" height="341" alt="1_LwHnEAaGZIjBJYJIAjdOsw" src="https://github.com/user-attachments/assets/ce7aa3cf-d452-4af1-894f-4dd194edef41" />

- Provide the existing token for the project and click on Continue

<img width="720" height="340" alt="1_a8A-Jm-WNHkckqhQzX5nDw" src="https://github.com/user-attachments/assets/73df2a33-e74e-41ff-874b-9fdd6c5d14b3" />

- Select JS/TS & Web option as our application is NodeJS-based, and copy the provided commands, as we will be using them in our Jenkins Pipelines

<img width="720" height="275" alt="1_Vn5sNyC58UWBQ70fZ1Vccw" src="https://github.com/user-attachments/assets/a8e75fdd-1749-40bb-b4a4-6b091b69a473" />

- Now, it’s time to create our Jenkins Pipeline
- Provide the name of your pipeline and select Pipeline as the Item type

<img width="720" height="394" alt="1_GA7J3DrLhDZ5DLUspdJGHQ" src="https://github.com/user-attachments/assets/208e94d6-b737-49bd-9df7-a9d63890b253" />

- We will be providing our Jenkins Pipeline Script from the GitHub Repo. Therefore, provide the information as written below

<img width="720" height="381" alt="1_Q1Mq8aD5H0Lv3TtT9xhIAQ" src="https://github.com/user-attachments/assets/358e3488-8d1e-412e-bba4-c5ca09808cf6" />

- If you see our Jenkins Pipeline has failed at OWASP DP Check, which means things are going as expected.
- Why does it fail at the OWASP DP Check? Because we did not install & configure OWASP DP on Jenkins.

<img width="720" height="207" alt="1_SH225wRIAv1wSXxOhNalxA" src="https://github.com/user-attachments/assets/45cb7552-42fb-449a-a895-4ace678daf35" />

- Here is the Sonar Scan for our Application Code

<img width="720" height="307" alt="1_8lthAzbRYP0wJx-nizOk8A" src="https://github.com/user-attachments/assets/3e77df58-315b-4a47-ab8f-bae1ecc89a06" />

- Also, I got the notification on our Slack channel

<img width="720" height="122" alt="1_vyTckK9loL8lZEJ3KgNO6A" src="https://github.com/user-attachments/assets/804b9e5e-d085-438e-9062-8fb671281849" />

- Go to Manage Jenkins -> Plugins and look for OWASP, and download the plugin

<img width="720" height="154" alt="1_wqnVDZVZ17ZCjAAlbzoEQA" src="https://github.com/user-attachments/assets/c55fbfc1-43e5-43bd-949b-0b6a79758dc5" />

- Once you install the plugin, configure the OWASP DP Check tool
- Go to Manage Plugins -> Tools and look for Dependency-Check
- Make sure to select the Install Automatically and should be GitHub

<img width="720" height="303" alt="1_Gv1NXewWVJmx-eIDSzzCJA" src="https://github.com/user-attachments/assets/613b9956-6584-45bd-8a77-b9df2a604206" />

- Now, run the pipeline again

<img width="720" height="322" alt="1_2AY9dEgTJ-sFM2hc0-4-lA" src="https://github.com/user-attachments/assets/64174b6c-3b29-4cb3-85e8-62dc08dcc423" />

- Pipeline is Successful for OWASP

<img width="720" height="244" alt="1_cycKRCm2gvaW5TdTSXkEnA" src="https://github.com/user-attachments/assets/b02c3fad-408b-4a7e-9845-22cdd9f4fa91" />

- We got the slack alert

<img width="720" height="242" alt="1_MC0MYkSa-7WXNZVohbp4Gg" src="https://github.com/user-attachments/assets/c3079306-eb91-45b2-8253-0201202ebe51" />

- Validate DP Check

<img width="720" height="220" alt="1_BREEJZRNxQNSxguFLkg3fA" src="https://github.com/user-attachments/assets/a2273125-c292-42c4-a9cb-53349eb95ec2" />

<img width="720" height="383" alt="1_g8QGIlilmzfQzytvElc93Q" src="https://github.com/user-attachments/assets/975ffb16-c13e-47b7-922f-08e418dfbc07" />

- As our Pipeline is failing, we have to set up Docker on our Jenkins
- We have to add one credential for our Docker Hub, as we will be pushing our Docker images to Docker Hub
- For that, we will have to generate a Personal Access Token

<img width="720" height="243" alt="1_xlylbq8UD9UtetIwV3AS7w" src="https://github.com/user-attachments/assets/0209c26b-cf8f-4a61-810c-da006dcbbdcb" />

- Once you have generated the PAT, add it to Jenkins

<img width="720" height="393" alt="1_ZNGb9-OeVRU7kleqlLWzAA" src="https://github.com/user-attachments/assets/7b0d9656-3a81-409a-a46b-3cf4a18cd94f" />

- Now, install the Docker plugin for building the images with Jenkins

<img width="720" height="232" alt="1_Tbdk0dXR0JCiFmDkiuqKEA" src="https://github.com/user-attachments/assets/7a4f4e3a-1daf-4310-b117-ef2231278f80" />

- Once the plugin is installed. Now, we have to get the API Key to watch some Netflix movies. So we will use TMDB to get the API- https://www.themoviedb.org/
- You should create the account before using this.
- Go to the Settings under your profile on TMDB

<img width="720" height="340" alt="1_nLE5MVcmZByMNvGantmPJQ" src="https://github.com/user-attachments/assets/15d5a8b7-468c-4010-a496-40923474f8a7" />

- Click on API

<img width="720" height="245" alt="1_Dyj_NYnmQQli0GLsdc_Hhg" src="https://github.com/user-attachments/assets/878a3c98-ad21-4f8c-ad4a-48edf5d0c271" />

- Now, we have new API Key

<img width="720" height="302" alt="1_-2oU4_LKbHJdXIUM21kAaw" src="https://github.com/user-attachments/assets/3a383e4f-efc3-4543-955d-06b936b1fbec" />

- Now, we will have to create credentials in our Jenkins for the TMBD API key

<img width="720" height="336" alt="1_uB3NSmFTkDoqsgG8cgnW1w" src="https://github.com/user-attachments/assets/3066bd39-8213-4c3d-b4c2-b53c2fcf436b" />

- After adding the secrets, we have to configure the Docker tool

<img width="720" height="379" alt="1_w2G2ySs7yjxoc0oL2zKg6Q" src="https://github.com/user-attachments/assets/5957e4ec-633d-40b2-b5aa-544d3bf2b591" />

- Now, run the pipeline again

<img width="720" height="290" alt="1_wbglNVyuO0U6WnhH6Ttmmg" src="https://github.com/user-attachments/assets/3c2e970e-7c8a-44f3-ac77-e2fae7b95d4c" />

<img width="720" height="229" alt="1_Ce7Kj1irJdndFKTZqsaCiQ" src="https://github.com/user-attachments/assets/4555ec55-9098-434c-8fc4-0538dbb842b6" />

- You can see our Docker Image has been pushed to Docker Hub

<img width="720" height="180" alt="1_bUfCR99vXRf69KgMob_BOA" src="https://github.com/user-attachments/assets/594620a5-00e1-4f98-b86e-813e3ba8d327" />

### Conclusion
- By now, you’ve transformed Jenkins from a static CI tool into a full-blown DevSecOps automation machine.
- Your pipeline scans, builds, and packages your app like a pro — it just needs a Kubernetes cluster to complete the loop.

## Deploying Applications on AWS Unmanaged Kubernetes Cluster

### Introduction
- we’ll go beyond automation and dive into the orchestration world by setting up an unmanaged Kubernetes cluster on AWS with one Master Node and one Worker Node.
- This is where your Jenkins pipeline stops at “Deployment Failed” and starts saying "Deployment Successful."
- Welcome to the part where your Netflix Clone finally comes to life — running inside containers, orchestrated by Kubernetes.

### Objective
- By the end of this part, you’ll:
  - Set up an unmanaged Kubernetes cluster with Master and Worker nodes on AWS EC2.
  - Connect your Jenkins pipeline with the cluster using kubectl and service accounts.
  - Deploy your Netflix Clone microservice from Jenkins to Kubernetes.
  - Fix the previous deployment failure and watch your CI/CD pipeline complete successfully.
- This part bridges the gap between infrastructure and application — where automation meets orchestration.

### Hands-On

### Set up a Kubernetes Cluster using Kubeadm
- Log in to the Master node and set the hostname
```bash
sudo hostnamectl set-hostname k8s-master
```
<img width="720" height="113" alt="1_f_6nnaSrAMf6-TxpzYvasQ" src="https://github.com/user-attachments/assets/1355a2a5-398c-4dfe-99f2-b32ccdc3f8fa" />

- Log in to the Worker node and set the hostname
```bash
sudo hostnamectl set-hostname k8s-worker
```
<img width="720" height="111" alt="1_3BmUFvSXBwrU8-S0EEizSg" src="https://github.com/user-attachments/assets/75653cd8-e573-49b7-828e-f817d55cbd69" />

- Both Node
```bash
export K8S_VER="1.33.5-00"
export KUBEADM_K8S_VERSION="v1.33.5"
```
<img width="720" height="42" alt="1_PKmf6vOgWXjiE6Dh5LsHFQ" src="https://github.com/user-attachments/assets/f89236b9-576d-4047-a10a-1a7a9db2ecb6" />

- Both Node
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

```bash
# load required kernel modules
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

```bash
# sysctl params required by k8s
cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

```bash
# apply sysctl params
sudo sysctl --system
```

<img width="720" height="310" alt="1_-rVotvuC8-zNWchy12rUjg" src="https://github.com/user-attachments/assets/6a56b77a-ead3-4248-9e4a-93e03fd61d75" />

- Both Node
```bash
# Install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release apt-transport-https
```

```bash
# Add Docker/Containerd repo (using official Docker repo for containerd)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
sudo apt update
sudo apt install -y containerd.io
```

```bash
# Configure containerd and enable systemd cgroup driver
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
# Ensure SystemdCgroup = true (line may exist as "SystemdCgroup = false" - make deterministic)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

```bash
# restart & enable
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd --no-pager
```

<img width="720" height="192" alt="1_pC66Ulv3jitXsAqNiJVTMw" src="https://github.com/user-attachments/assets/bb9763af-aa4d-41dc-b8b3-32c3970fa6f2" />

- Both Node
```bash
sudo su
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | \
tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
apt update
```

<img width="720" height="231" alt="1_BMN6YUZYUMD19WBopG3gSQ" src="https://github.com/user-attachments/assets/983d1bc4-6f80-4b58-93e3-914f4d3cf58f" />

- Both Node
```bash
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

<img width="720" height="316" alt="1_EKxyAHXGEawwri81h70ccA" src="https://github.com/user-attachments/assets/b897e203-d652-469b-a45b-f7d7ad39833c" />

- On the Master node only
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

<img width="720" height="366" alt="1_JoFddJdMItdJYNrijz0kCQ" src="https://github.com/user-attachments/assets/92d3b940-c79f-4d4f-9d2f-3f302793ea60" />

- On the Master node only
```bash
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

<img width="720" height="29" alt="1_wiSgq15_R9ohmU1QFUE36g" src="https://github.com/user-attachments/assets/a6378321-218f-494b-92e3-8b3119788610" />

- On the Master node only
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

<img width="720" height="76" alt="1_E3lTtXumDJZ9TQo8o6Xpbg" src="https://github.com/user-attachments/assets/6e3dd7fb-7348-4aac-8280-9279c5c86c89" />

- On the Master node only
```bash
kubectl get pods -A
```

<img width="720" height="94" alt="1_xkLkwG7gJ1mELevaNR3b5g" src="https://github.com/user-attachments/assets/d4ce8f2f-dff2-43b9-ab49-1cbeab3d3003" />

- On the Master node only
```bash
kubectl get nodes
```

<img width="720" height="63" alt="1_BtWJ1JBMy3WslRRvKwduUQ" src="https://github.com/user-attachments/assets/4d5efa69-df3e-4962-b292-1ee64f343256" />

- On the Worker node only
```bash
kubeadm join 10.0.35.70:6443 --token 8paztu.<token> \
        --discovery-token-ca-cert-hash sha256:<digest>
```

<img width="720" height="160" alt="1_jQCa5VhsoO35CfMcqtVEPA" src="https://github.com/user-attachments/assets/a8bca018-9026-4aff-8acf-ca735c6698c8" />

- Integrate K8S with Jenkins

<img width="720" height="332" alt="1_ESiCIU0pCUXk9LMObFn1Aw" src="https://github.com/user-attachments/assets/d6b161fd-a451-4bbf-be87-02c273e492bb" />

```bash
sudo cat /etc/kubernetes/admin.conf
```

<img width="720" height="156" alt="1_s6WMbLTpyLkx4hlitImS8w" src="https://github.com/user-attachments/assets/70062691-3a75-4514-9a14-04ee8f4865ee" />

<img width="720" height="345" alt="1_n-1iJUL3s6P8K3-VcPVarA" src="https://github.com/user-attachments/assets/a089db64-4b63-454d-9e3b-ec25100217f8" />

- Search
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

```bash
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

```bash
# Apply sysctl params without reboot
sudo sysctl --system
```

<img width="720" height="253" alt="1_WhpZPyHe4QFxL_cvOKlp-g" src="https://github.com/user-attachments/assets/f3d36707-428f-4cb6-984a-88db58d950ae" />

```bash
sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```

<img width="720" height="29" alt="1_oNt3WL86HjCoKQeJkP33uw" src="https://github.com/user-attachments/assets/5f75095c-a0fc-4f39-a9bc-c9a44f7f46c4" />

```bash
# Root User
sudo so
# Kuernetes Variable Declaration
KUBERNETES_VERSION=v1.33
CRIO_VERSION=v1.33v
```

```bash
# Apply sysctl params without reboot
sudo sysctl --system
```

```bash
sudo apt-get update -y
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

```bash
## Install CRIO Runtime
sudo apt-get update -y
apt-get install -y software-properties-common curl apt-transport-https ca-certificates
```

```bash
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list
```

```bash
sudo apt-get update -y
sudo apt-get install -y cri-o
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service
```

```bash
echo "CRI runtime installed susccessfully"
```

<img width="720" height="336" alt="1_XYoPYw4R8G_w0pJN0fnyvQ" src="https://github.com/user-attachments/assets/737be250-0723-451c-b881-64083e816a10" />

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

<img width="720" height="64" alt="1_DT-F6AmeY926ydA72yI5kw" src="https://github.com/user-attachments/assets/6619ab24-fb35-4f42-bd30-4c6fdea0e387" />

```bash
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
```

<img width="720" height="220" alt="1_OGdci6nAfyUYnxEcMgPqWw" src="https://github.com/user-attachments/assets/510849a1-e834-44b3-9f40-4f893c575aac" />

```bash
systemctl restart kubelet.service
systemctl enable kubelet.service
```

- Re-run the pipeline.

<img width="720" height="395" alt="1_-xJeycq0KrPFqgYIgpxMkA" src="https://github.com/user-attachments/assets/fd249c1b-531f-44bd-9c88-3f27bbb670a6" />

- Check the Pods and all resources of Kubernetes
```bash
kubectl get all -n default
```

<img width="720" height="150" alt="1_AXvnL9UTaH22FeE8VxWTig" src="https://github.com/user-attachments/assets/36f2fa8c-7680-4951-9bdf-18328f9e251c" />

- Access the Application

<img width="720" height="416" alt="1_hiGQC28LKZsO5agc8CvCMg" src="https://github.com/user-attachments/assets/e32c6ad9-d677-43d7-b16f-406161515adf" />

### Conclusion — What’s Next
- Your app’s now alive inside Kubernetes — scalable, containerised, and running smoothly.
- We’ve officially completed the Dev + Sec + Ops integration pipeline from code to cluster.
- We’ll shift gears to Monitoring and Observability — setting up Prometheus, Grafana, and Node Exporter to keep an eye on everything:
  - Jenkins
  - Kubernetes (Master + Worker)
  - The monitoring server itself

## Monitoring Jenkins and Kubernetes with Prometheus & Grafana

### Introduction
- we move from creation to observation.
- This is where you’ll set up a full-fledged Monitoring and Alerting Stack for your DevSecOps ecosystem — so you can actually see your Jenkins jobs, Kubernetes workloads, and servers doing their thing in real time.

### Objective
- By the end of this part, you’ll have:
  - A dedicated Monitoring Server running Prometheus and Grafana
  -  Node Exporter configured on Jenkins, Kubernetes Master, and Worker nodes.
  -  Kube State Metrics is integrated to monitor the entire K8S cluster.
  -  Real-time dashboards tracking system health, resource usage, and performance trends.
- In short, you’ll set up a monitoring system that not only shows what’s running but also warns you what’s about to break.

### Hands-On

- Update the server
```bash
sudo apt-get update
```

<img width="720" height="75" alt="1__FFiXnMWBwfOFU578l1JPA" src="https://github.com/user-attachments/assets/06b9ddb5-54e7-4331-baa8-1be468310b9a" />

- Create a system user for Prometheus
```bash
sudo groupadd --system prometheus
sudo useradd -s /sbin/nologin --system -g prometheus prometheus
```

<img width="720" height="73" alt="1_rx6wG_nI6Sc_KSxvocPi0w" src="https://github.com/user-attachments/assets/81c56447-b4b1-467e-afd8-cf28b8c6bb94" />

- Create a directory for Prometheus
```bash
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```

<img width="720" height="76" alt="1_RUDo2EW_Ls40zWywFyajrw" src="https://github.com/user-attachments/assets/c31bf3d3-11d4-44c4-939d-5b99d7dbbc1a" />

- Install the Prometheus binary
```bash
https://github.com/prometheus/prometheus/releases/download/v3.6.0/prometheus-3.6.0.linux-amd64.tar.gz
```

<img width="720" height="292" alt="1_-yMMIVvQZj9MbBfQtz4nmw" src="https://github.com/user-attachments/assets/72131c2a-4b1a-4b3e-964e-a9b384ab3707" />

- Extract the binary using the command below
```bash
tar vxf prometheus*.tar.gz
```

<img width="720" height="166" alt="1_w5QJzRmbaJzoNqoFBGATPg" src="https://github.com/user-attachments/assets/489c17ce-a543-434b-8a1f-0b95f78fd90a" />

- Navigate to the Prometheus directory
```bash
cd prometheus*/
```

<img width="720" height="46" alt="1_vTGYDgfL8df254sAzukCJg" src="https://github.com/user-attachments/assets/6e9a0dbb-241d-4eef-b46b-05244f0d7fa7" />

- Move the prometheus and promtool binary files to the /usr/local/bin directory and update the ownership
```bash
sudo mv prometheus.yml /etc/prometheus/
sudo chown prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

<img width="720" height="77" alt="1_QMSEMDISgPPjVsCuVb_tWA" src="https://github.com/user-attachments/assets/aa7ef5b8-05a1-4baf-9f0d-ffb666923633" />

<img width="720" height="70" alt="1_Ic3hG_1DU5QC0C1cMTjOZw" src="https://github.com/user-attachments/assets/d8a2a0cc-5975-46e8-9c5f-14721f743a81" />

- Create the Prometheus Systemd Service
```bash
sudo nano /etc/systemd/system/prometheus.service
```

```yaml
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
 --config.file=/etc/prometheus/prometheus.yml \
 --storage.tsdb.path=/var/lib/prometheus/ \
 --web.console.templates=/etc/prometheus/consoles \
 --web.console.libraries=/etc/prometheus/console_libraries
[Install]
WantedBy=multi-user.target
```

<img width="720" height="364" alt="1_0MouXXjjiFjqgbP6Hj1uXQ" src="https://github.com/user-attachments/assets/56106809-baaf-4d33-addf-f9150244a519" />

- Start the Prometheus service
```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

<img width="720" height="64" alt="1_9s6itCnOqXurnI2CWc-Rqw" src="https://github.com/user-attachments/assets/ff0fddf8-92c5-4125-b67f-251fd45f2cf8" />

- Check the status of the Prometheus service
```bash
sudo systemctl status prometheus
```

<img width="720" height="201" alt="1_4Vtu53oz8yd2Yrtp9tRNdQ" src="https://github.com/user-attachments/assets/126a7404-ffb7-4bf0-ac75-622afd0ce71a" />

- Access the Prometheus UI

<img width="720" height="222" alt="1_ASDskEGverM1pkMvqaYLJg" src="https://github.com/user-attachments/assets/8a190826-3b91-493f-ad04-a664bc421c51" />

- Download Node Exporter package
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
```

<img width="720" height="274" alt="1_PGeEvPRuIyCaUU6vLbGQKg" src="https://github.com/user-attachments/assets/04d713c4-c2f3-4259-bf55-9f09a4b960ec" />

- Unzip the node exporter file
```bash
sudo tar xvfz node_exporter-*.*-amd64.tar.gz
```

<img width="720" height="83" alt="1_jzTl4E-pL_RiitVKNR5DkQ" src="https://github.com/user-attachments/assets/9d423126-5860-4a72-94c8-be2f6507715e" />

- Create the node_exporter user
```bash
sudo useradd -rs /bin/false node_exporter
```

- Move the package and provide the necessary permissions
```bash
sudo mv node_exporter-*.*-amd64/node_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

<img width="720" height="148" alt="1_Pmpt2GNRqZ9BbSv7NsCJ4w" src="https://github.com/user-attachments/assets/9b531549-ce85-4676-8751-ba4851f5be98" />

- Create a Node Exporter systemd service
```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```yaml
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

<img width="720" height="194" alt="1_By8gPEhwasIRtDcH1idHxg" src="https://github.com/user-attachments/assets/237e13a5-7e8a-4093-ad9b-f3bc0c2205ea" />

- Enable the Node Exporter service and start the service
```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```

<img width="720" height="194" alt="1_By8gPEhwasIRtDcH1idHxg" src="https://github.com/user-attachments/assets/ad01f6aa-1975-4cd7-9740-2a7958b923a6" />

- Edit the Prometheus service and node exporter to scrape its metrics
```bash
sudo vim /etc/prometheus/prometheus.yml
cat /etc/prometheus/prometheus.yml
```

```yaml
- job_name: 'Node_Exporter'
  scrape_interval: 5s
  static_configs:
    - targets: ['34.200.245.177(MonotoringServerIP):9100']
```

<img width="720" height="210" alt="1_48mOC9_IW3YvFVP6jRzjUw" src="https://github.com/user-attachments/assets/d164bc3e-b1fe-4b26-a737-adaab14a1b89" />

- As we made some changes in the configurations, we have to restart the Prometheus server
```bash
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

<img width="720" height="128" alt="1_inpoc4xm9U9k8mYDAm3Kbw" src="https://github.com/user-attachments/assets/140515c8-c92a-4d22-aa7f-06999c020294" />

- Check the Prometheus Dashboard

<img width="720" height="231" alt="1_yad5JBzeu2MGtQGiNqBcYQ" src="https://github.com/user-attachments/assets/7d5fe148-e6d1-49c9-b98d-aeb20bb2c413" />

- Now, we can validate the metrics using the curl command
```bash
curl http://34.200.245.177:9100/metrics
```

<img width="720" height="131" alt="1_0fZjdqYj2uPwljoRQsaP6g" src="https://github.com/user-attachments/assets/47c73a3c-f328-48a4-ba95-5f6f1dd4c122" />

- Configure the Grafana key
```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```

<img width="720" height="34" alt="1_ClrRdF5L6Bp6ulSEIlvTFA" src="https://github.com/user-attachments/assets/f67a8c65-5012-4de9-87a6-04400a701806" />

- Configure the Grafana repo
```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
# Updates the list of available packages
sudo apt-get update
```

<img width="720" height="130" alt="1_OggMTEWse-Re-7WftdLOwQ" src="https://github.com/user-attachments/assets/1c64f358-e964-4843-8a2d-3d2aee6b154d" />

- Install Grafana using the command below
```bash
sudo apt-get install grafana -y
```

<img width="720" height="225" alt="1_eBssCUPrBTCuVcQBo2zJdQ" src="https://github.com/user-attachments/assets/018faba2-e8b9-4603-bc3e-a89035d525fa" />

- Now, enable the grafana service and start it
```bash
sudo systemctl enable grafana-server.service
sudo systemctl start grafana-server.service
sudo systemctl status grafana-server.service
```

<img width="720" height="175" alt="1_EOMTdYqxfliwvjHdy836_A" src="https://github.com/user-attachments/assets/a0d274bd-e136-4c9a-8355-954f7464ab95" />

- Now, we access the Grafana UI

<img width="720" height="418" alt="1_DUEZ2MZ-aghw-Q8ChL9JJg" src="https://github.com/user-attachments/assets/2a6c9870-05b7-4647-b0ec-1ad2b857a6af" />

- Here we can see our Grafana dashboard

<img width="720" height="395" alt="1_ZMJJKl5DKA8Mo8cW-tTFMQ" src="https://github.com/user-attachments/assets/330bb9e5-8f46-4387-a64f-ba495328d16f" />

- Click on Connections and go to Data sources

<img width="720" height="262" alt="1_X-BI2lWZUWxKeRsz8LDiqA" src="https://github.com/user-attachments/assets/f8a87c0b-910f-485b-a542-974ae900961f" />

- Select Prometheus as data source

<img width="720" height="282" alt="1_Nkx_i4gcPTh6XowW4tUPjA" src="https://github.com/user-attachments/assets/1bce88a4-420a-498a-8df2-ecb3bc3a0421" />

- Now, provide the correct Prometheus server URL

<img width="720" height="370" alt="1_nE45nnOfy2eHJ5jwDM9nBQ" src="https://github.com/user-attachments/assets/6f8d2259-ae57-454b-9a67-6048d9833675" />

- Now, Data Source has been added

<img width="720" height="232" alt="1_RJdFgl2wh20B0eYMZ6adXg" src="https://github.com/user-attachments/assets/6db9185b-ad5f-4895-9f9c-f4ecc2388567" />

- Now, we have to import the dashboard.
- Click on New and select Import dashboard

<img width="720" height="308" alt="1_yl9WO1dXnl75uJQWY4iSKA" src="https://github.com/user-attachments/assets/f58bf6ac-61fd-4d86-bcd9-6da0c2471092" />

- Add the ID 14513 to view the Linux-based metrics on the Grafana dashboard

<img width="720" height="364" alt="1_dTlV412Kct-0ZuGZguDgag" src="https://github.com/user-attachments/assets/624ad177-05f7-4aee-9ba2-dfb40d861e21" />

- Select the correct Prometheus source and click on Import

<img width="720" height="365" alt="1_pTVvmRtpWvNoxZXDli4acg" src="https://github.com/user-attachments/assets/3c6e65c1-af72-4b17-a6ce-3818d44f73a6" />

- Now, we can use the Grafana Dashboard for the metrics

<img width="720" height="394" alt="1_r0EMXroeOqxMvLO4m7l4Ug" src="https://github.com/user-attachments/assets/a6256fef-fded-4fa1-ae33-7149d124008a" />

- Now, we will enable monitoring for Jenkins
- Install the plugin first

<img width="720" height="154" alt="1_d69rzxe4KkpNKD4bxQpY9w" src="https://github.com/user-attachments/assets/a4e0e20c-9255-407c-8db1-60b3b6c7fe3a" />

- Now, we have to set node_exporter as a target for Prometheus.
- For that, edit prometheus.yaml located at /etc/prometheus/prometheus.yaml
```bash
sudo vim /etc/prometheus/prometheus.yml
```
```yaml
- job_name: "jenkins"
    static_configs:
      - targets: ["54.224.137.154:8080"]
```

<img width="720" height="219" alt="1_0_BpcCeYOD3oDwqdS2emTg" src="https://github.com/user-attachments/assets/7fa4ddf2-5212-4353-b306-6ea493cfd9e0" />

```bash
promtool check config /etc/prometheus/prometheus.yml
```

<img width="720" height="61" alt="1_taFhSHFOe7EHaj0ms-AKOA" src="https://github.com/user-attachments/assets/59fa326a-81c3-4585-81bb-9b666f4f5ba9" />

- Now, we can validate the targets from Prometheus UI as well

<img width="720" height="315" alt="1_77mXcncgbn5b2MYsKCs1GA" src="https://github.com/user-attachments/assets/c409525e-b220-41c9-b445-80942ef87877" />

- Now, we will import the dashboard into Grafana

<img width="720" height="356" alt="1_fW9-lxkFdf1-sHOZ5L0cxg" src="https://github.com/user-attachments/assets/b5dad5cd-e133-44f2-a6e9-b9a3991951fc" />

- Now, we can see the dashboard

<img width="720" height="393" alt="1_0hzPOQg40MiOs_sr6KGp7g" src="https://github.com/user-attachments/assets/0c56a0eb-19bf-44fa-b125-bf1efe085da6" />

### Monitoring on Kubernetes Cluster
- Now, we have to run this on both Nodes, including the Master and Worker
- Create a node exporter user to run it
```bash
sudo useradd -rs /bin/false node_exporter
```
- Download the Node Exporter binary
- Unzip the Node Exporter package
- Move the binary file to the /usr/local/bin directory

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz 
tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz 
sudo mv node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
```

<img width="720" height="348" alt="1_ZpDycO9rlzyYyQbWJW9r3w" src="https://github.com/user-attachments/assets/9870e910-0e90-4344-9031-4ec1c0c0f26b" />

- Now, create a node exporter systemd service
```bash
sudo nano /etc/systemd/system/node_exporter.service
```
```yaml
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

<img width="720" height="194" alt="1_By8gPEhwasIRtDcH1idHxg" src="https://github.com/user-attachments/assets/183bd226-4605-414a-92b8-8f6a4b9a7d32" />

- Now, we have to start the node exporter service using the commands below
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

<img width="720" height="243" alt="1_EldpXqefLJbgZ0KesRT4ow" src="https://github.com/user-attachments/assets/54ff96b5-5cd4-4e29-a796-05095cd7eb85" />

- Add a target to the Prometheus server(Monitoring)
```bash
sudo vim /etc/prometheus/prometheus.yml
```
```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

 # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["localhost:9090"]
       # The label name is added as a label `label_name=<label_value>` to any timeseries scraped from this config.
        labels:
          app: "prometheus"
  - job_name: 'Node_Exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['34.200.245.177:9100']
  - job_name: "jenkins"
    static_configs:
      - targets: ["54.224.137.154:8080"]
    metrics_path: "/prometheus"
  - job_name: "node_exporter_k8smaster"
    static_configs:
      - targets: ["35.175.234.123:9100"]
  - job_name: "node_exporter_k8sworker"
    static_configs:
      - targets: ["98.80.118.244:9100"]
```

<img width="720" height="290" alt="1_rsBxPpbdSXHoSueu91pMbg" src="https://github.com/user-attachments/assets/c0036f13-858e-40c2-982d-4382c6e2f5ae" />

- After adding the configuration, we have to restart Prometheus
```bash
sudo systemctl restart prometheus.service
sudo systemctl status prometheus.service
```

<img width="720" height="214" alt="1_Vg5Tk1WZ_rlB5F62EqQb9Q" src="https://github.com/user-attachments/assets/0e1aaf22-44bc-4e43-8613-b7f564245985" />

- Now, we will validate from Prometheus Targets

<img width="720" height="352" alt="1_ndw4ahzjQi4QbT-UzuSFYA" src="https://github.com/user-attachments/assets/68fa688b-ed25-4166-b9a3-229e2d1fe139" />

- To see the metrics, click on the existing dashboard Linux Exporter Node

<img width="720" height="149" alt="1_Cqd4uRZC3d8NwxBDzx0q6A" src="https://github.com/user-attachments/assets/bcf2cb06-2fa0-4eae-850b-eee75e78f9cc" />

- Monitoring for Master Node

<img width="720" height="337" alt="1_5IwH1-Mua0UFaTvVjwdKLw" src="https://github.com/user-attachments/assets/b4d76f85-523e-4500-936b-9f3de542b312" />

- Monitoring for Worker Node

<img width="720" height="339" alt="1_UQuteKuebn7QWAte2qRjkQ" src="https://github.com/user-attachments/assets/df9d0a28-bc8b-4842-8ce4-6ecebce93d18" />

- Monitoring for Jenkins Server

<img width="720" height="394" alt="1_sBR8dFqBKxSEn8zCz66hwQ" src="https://github.com/user-attachments/assets/ef29002d-797d-4622-81ac-c9457f0aa9b0" />

- Monitoring Server

<img width="720" height="394" alt="1_oRHDoWZt4wjhYxsRfyLuuA" src="https://github.com/user-attachments/assets/77b3bda0-f5a3-441b-b0a9-4577069e2d5f" />
