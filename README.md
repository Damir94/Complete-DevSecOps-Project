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

<img width="720" height="206" alt="1_T-QRcU9Bj7Kt_i8gloQDow" src="https://github.com/user-attachments/assets/2cf73edd-fd6d-41bf-a2b6-ee8081ebb2e4" />

- Log in as the ubuntu user
```bash
sudo su ubuntu
cd
```

<img width="720" height="129" alt="1_1DAL0Yv7ZRIYpaaj6Rxz8Q" src="https://github.com/user-attachments/assets/4eafa21f-c157-4bce-9860-cd45ad005d2d" />

- To install Jenkins , we need to install Java first.
```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
java --version
```

<img width="720" height="295" alt="1_OzDJwN1tzAjtVhNtZgB6Ag" src="https://github.com/user-attachments/assets/02e4cd3e-880f-4fe9-8dd6-aaa4a2b118a1" />

- Now, we will be going to install jenkins
```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
```

<img width="720" height="367" alt="1_pcznsEufD2V_DVmyemufGg" src="https://github.com/user-attachments/assets/1288db09-d9d1-45d9-aa20-32f26e71440a" />

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

<img width="720" height="418" alt="1_en8FQAVsuBHCIvepAqYZKQ" src="https://github.com/user-attachments/assets/08189826-c09d-495b-836d-e8ed2f6ef842" />

- Click on Save and Finish

<img width="720" height="417" alt="1_ezyEv2uFWgA085zs2kIfQw" src="https://github.com/user-attachments/assets/03aed7a1-ee45-4da2-8eed-18bdc32af92f" />

- We are ready with our Jenkins Server

<img width="720" height="391" alt="1_CspSPHwSgYbvgTbcwMCHUg" src="https://github.com/user-attachments/assets/19296b79-02cb-4685-8764-faec48825078" />

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

<img width="720" height="287" alt="1_O1mLDBcVSd1ST2aSNYd6oA" src="https://github.com/user-attachments/assets/b34d2d3e-417b-4466-9138-905a0c5be5c5" />

- After installing Docker, we will be going to install Sonarqube for Code Quality.
- We have two options to install Sonarqube which are installing Sonarqube on EC2 directly or installing Sonarqubeusing Docker container
- We will be using docker to install it Sonarqube
```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:community
```
<img width="720" height="181" alt="1_YTFPIHZt_WLAhTjrB1LbUg" src="https://github.com/user-attachments/assets/fdda6209-c48a-4a42-8316-f40fb7b0ade5" />

- The Sonarqube docker container is running, and now we will access it using the Same Jenkins IP with Port 9000
- To access SonarQube, the username and password are admin

<img width="720" height="355" alt="1_1C8cHaQ4Hkif7FLCU4a_yg" src="https://github.com/user-attachments/assets/77b42bc9-25b1-45fa-8820-d8a6354cbe38" />

- Update the password of your SonarQube

<img width="720" height="244" alt="1_YHOpkOpdzxxGENNO2sa8Ng" src="https://github.com/user-attachments/assets/c2891141-0aed-4c9d-a4f9-c42300e99dab" />

- Here is the SonarQube dashboard

<img width="720" height="350" alt="1_3nZ71-U96z6t03JAlPnc9A" src="https://github.com/user-attachments/assets/e19c79bb-092a-4b44-87b6-cc2854900e8e" />

- Now, we have to install our next tool, which is trivy
```bash
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
trivy --version
```

<img width="720" height="273" alt="1_8JLP2lBPbuWrHGKZg8rabA" src="https://github.com/user-attachments/assets/637369a7-8f2b-43e9-8f97-b6604f37fce2" />

- Now, we will install kubectl utility
```bash
curl -LO https://dl.k8s.io/release/v1.33.5/bin/linux/amd64/kubectl
curl -LO https://dl.k8s.io/release/v1.33.5/bin/linux/amd64/kubectl.sha256
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
<img width="720" height="157" alt="1_VTTPjX7dRfTvoQHbFeGZnw" src="https://github.com/user-attachments/assets/eaa2b28c-b7eb-42b9-8fbf-ace0d4987c73" />

- Now, we will be setting up Slack for notification of our Pipelines whenever they finish with any status like Failed, Succeeded, or Aborted.
- For that, we have to install the Jenkins plugin called Slack Notification

<img width="720" height="144" alt="1_dUNgAOmL_iuZnCGNGZS3MQ" src="https://github.com/user-attachments/assets/9d28bca3-df41-4c91-ab9e-5e1ba34eb015" />

- Plugin is installed. Check the Restart Jenkins options

<img width="720" height="343" alt="1_FHeIQKki8x9EG9EL9trGeQ" src="https://github.com/user-attachments/assets/76ee88fb-6d82-4d5d-917e-1954b036f8ba" />

- Now, we have to configure the authentication from the Slack side
- Go to the Link: https://api.slack.com/apps and make sure you have already signed in with your official Slack account
- Click on Create New App

<img width="720" height="223" alt="1_M5ssCvCs-4hmiYnVTBbrDA" src="https://github.com/user-attachments/assets/f11aa5ff-89e2-41c7-abb6-ec5f2130956a" />

- We will be creating an app using a manifest. So, click on it

<img width="720" height="259" alt="1_sTD08oEMCzqjjGGHMy-EUg" src="https://github.com/user-attachments/assets/c82d3ba7-876d-447d-9899-cd970ad2abd3" />

- Select the correct workspace for Slack

<img width="720" height="262" alt="1_NJLWZFi_f2bgG2go_kVj8Q" src="https://github.com/user-attachments/assets/48b8f285-aa6a-4481-af5b-edbcf3f72326" />

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

<img width="720" height="385" alt="1_-M13rOqP210HfrQQJUhlrQ" src="https://github.com/user-attachments/assets/1f0cbab5-6ee3-4b9c-9846-8e88c427dff5" />

- Review the provided information and click on Create

<img width="720" height="280" alt="1_8W6lm9-lbBtnIfw98TXXrg" src="https://github.com/user-attachments/assets/bceee056-365e-4df0-8f88-71522820f1fa" />

- After creating the app, your app will look like this

<img width="720" height="355" alt="1_MZzWv3joa0xFYISPfMSR8A" src="https://github.com/user-attachments/assets/73d9dc31-24c1-4d84-9f7b-b3aac34ac696" />

- You can configure the Display information

<img width="720" height="394" alt="1_6gGJ--6s8w7ImFIQXrW_JQ" src="https://github.com/user-attachments/assets/c67bec0f-612e-4745-be89-c6c52bfd940f" />

- Now, install the App on our Slack workspace
- Click on Install

<img width="720" height="205" alt="1_mgvLSob00JNwvHlZc6DyGw" src="https://github.com/user-attachments/assets/6996a015-4fce-4ccd-986b-941d29c1b624" />

- It will prompt you to reconfirm a few things. After confirming, click on Allow

<img width="720" height="354" alt="1_EHp9PqDcZime8oIQrvHYgw" src="https://github.com/user-attachments/assets/c4712998-783c-4a59-a185-f7df9fcbe907" />

- You will get the Bot User Token

<img width="720" height="242" alt="1_W3P6Otgq-_PGX-Wb_Cc1Ag" src="https://github.com/user-attachments/assets/e42c8efa-8096-4f6c-afb9-679ee9eaf065" />

- Also, if you check your Slack. You will see one app added called Jenkins

<img width="720" height="142" alt="1_4Azh78DWsSMXF1VBsa4VJg" src="https://github.com/user-attachments/assets/dfb62b66-8656-46eb-ac06-2a9abcacc12f" />

- Now, we have to copy the Bot User Token and use it in Jenkins
- Go to the Jenkins -> Manage Jenkins -> System and look for Slack

<img width="720" height="298" alt="1_z9DrRBG22B5CSZreHkaqKA" src="https://github.com/user-attachments/assets/d4077544-8909-4e68-9853-c79e438e2c3e" />

- We have to add a secret by clicking on +Add, which we copied in the earlier steps
- Click on Add

<img width="720" height="372" alt="1_czl_pg-tRRXb2EIJ31W8jg" src="https://github.com/user-attachments/assets/849d49ed-5570-499d-85eb-3b9924a90bbe" />

- After adding the secret, we have to invite the Jenkins app that we created in our Slack channel, as my channel is private. But if your channel is public, you don’t need to invite
```bash
/invite @Jenkins
```
<img width="720" height="261" alt="1_0iJFVZvh7b5UhO7jle3hRg" src="https://github.com/user-attachments/assets/808c2b87-8c64-42ad-9756-53b55bb95021" />

- Now, add the workspace name and channel name of your Slack where you want to get the notification of your Jenkins Pipeline and test the connection. As you can see, our Test Connection is successful.

<img width="720" height="290" alt="1_Ac4SK7K1bfsAK35J-vn2nw" src="https://github.com/user-attachments/assets/45577b0d-3055-4a9e-9659-d4da439bd8d5" />

- You can also validate from here your Slack channel as it sends you a ping

<img width="720" height="257" alt="1_wRZTMv5DWFgCczxzO_CB3g" src="https://github.com/user-attachments/assets/9ca17db3-07bc-4e8c-88aa-8b248a2df206" />

- Once Slack is working fine, then click on Save and Apply

<img width="720" height="378" alt="1_yomYx1uoTk5gvTL0Fbtslw" src="https://github.com/user-attachments/assets/2ad77f10-e475-4599-a6d6-3bd4e076cf06" />

### Conclusion
- With this setup, your DevOps engine is now fully functional and security-aware.
- You’ve got Jenkins running with Docker, SonarQube, and Trivy — ready to handle real workloads and enforce quality gates automatically.



