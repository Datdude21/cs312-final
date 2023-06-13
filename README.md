# CS 312 Final Project - Reese Clifford
* My submission utilizes Docker and ECS
## Requirements
1. Terraform
    * How to install on Windows
        * Open Console in Admin mode
        * Run 'choco install terraform'
        * Verify it's installed with 'terraform -help'
2. Docker 
    * How to install Docker on Windows
        * [Download Link Here](https://docs.docker.com/desktop/install/windows-install/)
        * When prompted, ensure the Use WSL 2 instead of Hyper-V option on the Configuration page is selected or not depending on your choice of backend.
        * Follow the instructions on the installation wizard to authorize the installer and proceed with the install.
        * When the installation is successful, click Close to complete the installation process.
        * If your admin account is different to your user account, you must add the user to the docker-users group. Run Computer Management as an administrator and navigate to Local Users and Groups > Groups > docker-users. Right-click to add the user to the group. Log out and log back in for the changes to take effect.
3. Github Repository
    * Log into your Github Account
    * Go to your account
        * Click Repositories
        * Then click New
        * Name Repository
        * Make public
        * Check add a README file
        * Create Repository
4. AWS CLI
    * How to install AWS CLI on Windows
        * Open console in admin mode
            * Run the following to download it 'msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi'
            * To confirm the installation run 'aws --version'
                * The following should show 'aws-cli/2.10.0 Python/3.11.2 Windows/10 exe/AMD64 prompt/off'



## Overview of Pipeline
1. Write an infrastructure provisioning script
    * Choose an infrastructure provisioning tool such as Terraform, Pulumi, or Ansible.
        * I used Terraform
    * Create a script file (e.g., minecraft.tf for Terraform) in your Git repository.
    * Write the necessary code to define and provision the required AWS resources for your Minecraft server (compute resources, networking, etc.).
        * Code will explained more in Tutorial section
    * Ensure that your script is properly configured to use the AWS credentials and region.
        * In your console type the following 'aws configure'. Four prompts should appear one at a time. Make sure you have the AWS CLI info from your starter lab up (AWS Details).
            * Copy & Paste the aws_access_key_id for the first one.
            * Copy & Paste the aws_secret_access_key for the second one.
            * Enter
            * Enter
        * Now type 'aws configure set aws_session_token " " ' Within the quotations past the token code.
        * You can now properly run your terraform script
2. Provision compute resources
    * Define and provision the compute resources required for your Minecraft server using the chosen infrastructure provisioning tool (e.g., EC2, ECS, EKS).
        * I did ECS for my project
    * Configure the instance(s) with the necessary specifications (e.g., instance type, disk size, security groups).
3. Set up networking
    * Configure the networking aspects required for your Minecraft server, such as setting up VPCs, subnets, security groups, and routing rules.
    * Ensure that the networking configuration allows inbound connections to the Minecraft server port (default: TCP port 25565).
4. Specify and configure the Docker image
    * If you choose to use Docker, specify and configure the Docker image for your Minecraft server.
    * Create a Dockerfile in your repository, specifying the necessary steps to build the image.
    * Configure any additional settings or environment variables required for the Minecraft server within the Dockerfile.
5. Connect to the Minecraft server
    * Once the provisioning and configuration are complete, obtain the public IP address of your task.
    * Connect to the Minecraft server using a Minecraft client by entering the server address (the public IP address).


## Tutorial 
### Writing the Terraform Script
#### 1. Region set up

    `provider "aws" {
        region = "us-east-1"
    }`
* Set the region

#### 2. VPC Module
    module "vpc" {
        source = "terraform-aws-modules/vpc/aws"
        name = "minecraft_vpc"
        cidr = "10.0.0.0/16"
        azs  = ["us-east-1a", "us-east-1b", "us-east-1c"]
        public_subnets = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
    }
* VPC is a virtual private cloud (VPC) that provides isolated network space for the Minecraft server.
* Cidr is the CIDR block to be used for the VPC.
* Azs are the availability zones to be used for the VPC public subnets.
* Public_subnets are the CIDR blocks to be used for the VPC public subnets.

#### 3. ECS Cluster 
    resource "aws_ecs_cluster" "minecraft_server" {
        name = "minecraft_server"
    }
* Setup and name for cluster
* ECS Cluster is an Amazon Elastic Container Service (ECS) cluster that manages the deployment and scaling of the Minecraft server.

#### 4. Security Group
    resource "aws_security_group" "minecraft_server" {
        name        = "minecraft_server"
        description = "minecraft_server"
        vpc_id      =  module.vpc.vpc_id
        
        ingress {
            description = "minecraft_server"
            from_port   = 25565
            to_port     = 25565
            protocol    = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
        }
        
        egress {
            from_port        = 0
            to_port          = 0
            protocol         = "-1"
            cidr_blocks      = ["0.0.0.0/0"]
            ipv6_cidr_blocks = ["::/0"]
        }
    }
* A security group that controls access to the Minecraft server. 
* Specifically the 'ingress' and 'egress' set the rules
    * Ingress allows incoming traffic on port 25565. 
    * egress allows traffic from all ports to join

#### 5. ECS Task Definition
    resource "aws_ecs_task_definition" "minecraft_server" {
        cpu                      = "4096"
        memory                   = "8192"
        family                   = "minecraft-server"
        network_mode             = "awsvpc"
        requires_compatibilities = ["FARGATE"]
        container_definitions = jsonencode([
            {
            name          = "minecraft-server"
            image         = "itzg/minecraft-server:java17-alpine"
            essential     = true
            tty           = true
            stdin_open    = true
            restart       = "unless-stopped"
            portMappings  = [
                {
                containerPort = 25565
                hostPort      = 25565
                protocol      = "tcp"
                }
            ]
            environment   = [
                {
                name  = "EULA"
                value = "TRUE"
                },
                {
                "name": "VERSION",
                "value": "1.20.1"
                }
            ]
            mountPoints   = [
                {
                containerPath = "/data"
                sourceVolume  = "minecraft-data"
                }
            ]
            }
        ])
        volume {
            name = "minecraft-data"
        }
    }
* An ECS task definition that specifies the resources to be used by the Minecraft server, including CPU, memory, and the Docker image to run.
* Docker Image
    * Under container_defintions.
    * Sets name, image, ports, restart, and version of Minecraft.
    * Also sets EULA to equal TRUE as the server would not be able to run. 

#### 6. ECS Service 
    resource "aws_ecs_service" "minecraft_server" {
        name            = "minecraft_server"
        cluster         = aws_ecs_cluster.minecraft_server.id
        task_definition = aws_ecs_task_definition.minecraft_server.arn
        desired_count   = 1
        network_configuration {
            subnets          = module.vpc.public_subnets
            security_groups  = [aws_security_group.minecraft_server.id]
            assign_public_ip = true
        }
        launch_type = "FARGATE"
    }
* An ECS service that manages the deployment and scaling of the Minecraft server tasks.
* This basically connects everything together.
    * Connects to the cluster, task definition, subnets, and security groups we made before.
    * It also assigns us a public ip address

#### 7. Connecting to the Minecraft Server
* Run 'terraform init'
    * Might need to update so run 'terraform init -upgrade'
* Run 'terraform fmt'
* Run 'terraform apply'
    * Type 'yes'
    * Wait until it is up
* Go to ECS on AWS console
    * Clusters > minecraft_server > Tasks
    * At tasks select the Networking tab
        * Copy the Public IP Address
* Run Minecraft
    * Make sure it is running version 1.20.1
    * Create a new server and paste the IP address
        * Join the Server

## Resoures/Sources

1. Github Respository 
    * I utilized the following repository for this project. I tried the EC2 route but I kept running into interference with Windows and Ansible. This repository utilizes Docker and ECS. I made some changes to make it run with the AWS verison we have and the Minecraft version. 
    * [Link to Repository](https://github.com/Yris-ops/minecraft-server-aws-ecs-fargate/blob/main/main.tf)