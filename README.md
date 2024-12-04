# Implement EKS Cluster with VPC Using Terraform for Real-time Scalable Infrastructure with Autoscaling Worker Nodes.

### ***Step 1: Setting Up a Virtual Machine (VM) to Run Terraform***
First we need a reliable environment to execute Terraform commands. I accomplished this by provisioning an EC2 instance with the following configuration:
- Instance Type: t2.micro (1 vCPUs, 1 GiB memory)
- Storage: 10 GiB of gp3 SSD
  
![Screenshot 2024-12-04 154743](https://github.com/user-attachments/assets/6ab4e4dc-dac0-41f4-a443-f35ac9d27799)


Once the EC2 instance was up and running, I performed the following steps:
#### 1. Updated the System and Installed AWS CLI and Terraform on the Machine.
``` shell
sudo apt update && sudo apt upgrade -y

# aws cli
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# install Terraform
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```
- Official Documentation of AWS: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- Official Documentation of Terraform: https://developer.hashicorp.com/terraform/install
#### 2. Created an IAM User with necessary permissions and generated access keys.
![Screenshot 2024-11-29 142535](https://github.com/user-attachments/assets/b96725e4-7677-4796-90e2-58f011e6186a)
![Screenshot 2024-11-29 142605](https://github.com/user-attachments/assets/ab59d9f8-e7b1-430b-a64b-7f13ca79b397)

#### 3. Configured AWS CLI with the access key, secret key, and region to enable secure access to AWS services for Terraform.
Finally, I configured the AWS CLI on the EC2 instance with the newly created access credentials:
``` shell
aws configure
```
![Screenshot 2024-11-29 143311](https://github.com/user-attachments/assets/61a3887d-2928-4b93-9fee-0a8c4f64416c)


### ***Step 2: Explaining Terraform "vpc.tf" configuration***
The vpc.tf file defines the configuration to create a Virtual Private Cloud (VPC) in AWS using Terraform. This VPC will serve as the networking backbone for an EKS (Elastic Kubernetes Service) cluster.

#### 1. AWS Provider Configuration
``` shell
provider "aws" {
  region = var.aws_region
}
```
- Purpose: Configures Terraform to interact with AWS services.
- region = var.aws_region: Specifies the AWS region where the resources will be created, using a variable (var.aws_region). This value must be defined in a variables.tf file or passed during execution.
#### 2. Data Source: AWS Availability Zones
``` shell
data "aws_availability_zones" "available" {}
```
- Purpose: Dynamically retrieves the list of available AWS Availability Zones (AZs) in the selected region.
- Usage: Used later to distribute subnets across multiple AZs for high availability.
#### 3. Local Variable: Cluster Name
``` shell
locals {
  cluster_name = "adi-eks-${random_string.suffix.result}"
}
```
- Purpose: Defines a local variable cluster_name to store the name of the EKS cluster.
- random_string.suffix.result: Appends a random string to ensure the cluster name is unique.
#### 4. Resource: Random String
``` shell
resource "random_string" "suffix" {
  length  = 8
  special = false
}
```
- Purpose: Generates an 8-character alphanumeric random string to ensure the cluster name and resources remain unique.
- special = false: Excludes special characters from the generated string.
#### 5. Module: VPC (Virtual Private Cloud)
``` shell
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.7.0"
```
- Source: Uses the official VPC module from the Terraform Registry (terraform-aws-modules/vpc/aws).
- Version: The module version is locked to 5.7.0 to ensure compatibility and stability.
- 5.1 VPC Configuration
``` shell
  name                 = "abhi-eks-vpc"
  cidr                 = var.vpc_cidr
  azs                  = data.aws_availability_zones.available.names
```
name: Assigns a name to the VPC (abhi-eks-vpc).
cidr: The IP range for the VPC, specified by the variable var.vpc_cidr.
azs: Distributes subnets across all available availability zones.
- 5.2 Subnet Configuration
``` shell
  private_subnets      = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets       = ["10.0.4.0/24", "10.0.5.0/24"]
```
Private Subnets:
IP ranges for two private subnets (10.0.1.0/24 and 10.0.2.0/24). These are used for internal resources (e.g., worker nodes).

Public Subnets:
IP ranges for two public subnets (10.0.4.0/24 and 10.0.5.0/24). These are used for resources that need internet access (e.g., load balancers).

- 5.3 NAT Gateway Configuration
``` shell
  enable_nat_gateway   = true
  single_nat_gateway   = true
```
enable_nat_gateway: Enables a NAT Gateway for internet access from private subnets.
single_nat_gateway: Specifies the creation of a single NAT Gateway (for cost efficiency).
- 5.4 DNS Settings
``` shell
  enable_dns_hostnames = true
  enable_dns_support   = true
```
enable_dns_hostnames: Enables DNS hostnames for instances in the VPC.
enable_dns_support: Enables DNS resolution for the VPC.
- 5.5 Tags for Resources
``` shell
  tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
  }
```
Purpose: Tags the VPC to indicate it belongs to a Kubernetes cluster.
Key: "kubernetes.io/cluster/${local.cluster_name}"
Value: "shared" – Indicates that the VPC can be shared with Kubernetes resources.
- 5.6 Public Subnet Tags
``` shell
  public_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                      = "1"
  }
```
Purpose: Tags the public subnets for use with external load balancers.
kubernetes.io/role/elb = 1: Marks these subnets as eligible for external load balancers.
- 5.7 Private Subnet Tags
``` shell
  private_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"             = "1"
  }
```
Purpose: Tags the private subnets for use with internal load balancers.
kubernetes.io/role/internal-elb = 1: Marks these subnets as eligible for internal load balancers.

### ***Step 3: Explaining Terraform "versions.tf" configuration***
The versions.tf file in Terraform is used to specify the version requirements for both Terraform and the providers it uses. It ensures compatibility between the infrastructure code, Terraform itself, and the providers that interact with external services like AWS or Kubernetes. Let’s break it down:

#### 1. Terraform Block
The terraform block defines the Terraform version and required providers.

``` shell
terraform {
  required_version = ">= 0.12"
  required_providers {
    ...
  }
}
```
- required_version = ">= 0.12"
- Specifies that the Terraform version must be 0.12 or higher.
- This ensures compatibility with modern Terraform syntax and features introduced after version 0.12.
#### 2. Required Providers
The required_providers block specifies the providers required by the Terraform configuration, including their source and version constraints.

- 2.1 Random Provider
``` shell
random = {
  source  = "hashicorp/random"
  version = "~> 3.1.0"
}
```
Purpose: The random provider is used to generate random values (e.g., random strings, passwords, or unique IDs).
Version Constraint:
~> 3.1.0 means:
Use any 3.x version that is greater than or equal to 3.1.0 but less than 4.0.0.
Example Use Case:
Generating a unique string for naming an EKS cluster:
``` shell
resource "random_string" "eks_name" {
  length  = 8
  special = false
}
``` 
- 2.2 Kubernetes Provider
``` shell
kubernetes = {
  source  = "hashicorp/kubernetes"
  version = ">=2.7.1"
}
```
Purpose: The kubernetes provider is used to interact with Kubernetes clusters, such as creating and managing resources like pods, services, or deployments.
Version Constraint:
>= 2.7.1 means:
Use version 2.7.1 or higher.
Example Use Case:
Deploying a Kubernetes service:
``` shell
resource "kubernetes_service" "example" {
  metadata {
    name = "example-service"
  }
  spec {
    selector = {
      app = "example-app"
    }
  }
}
```
- 2.3 AWS Provider
``` shell
aws = {
  source  = "hashicorp/aws"
  version = ">= 3.68.0"
}
```
Purpose: The aws provider allows Terraform to manage AWS resources, such as EC2 instances, VPCs, S3 buckets, and EKS clusters.
Version Constraint:
>= 3.68.0 means:
Use version 3.68.0 or higher.
Example Use Case:
Creating a VPC in AWS:
``` shell
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```
- 2.4 Local Provider
``` shell
local = {
  source  = "hashicorp/local"
  version = "~> 2.1.0"
}
```
Purpose: The local provider is used to manage local files and directories, and to generate local values.
Version Constraint:
~> 2.1.0 means:
Use any 2.x version that is greater than or equal to 2.1.0 but less than 3.0.0.
Example Use Case:
Generating a local file:
``` shell
resource "local_file" "example" {
  content  = "Hello, World!"
  filename = "hello.txt"
}
```
- 2.5 Null Provider
``` shell
null = {
  source  = "hashicorp/null"
  version = "~> 3.1.0"
}
```
Purpose: The null provider is used to define resources that do nothing but trigger other resources or outputs. It is often used in Terraform for testing or as a placeholder.
Version Constraint:
~> 3.1.0 means:
Use any 3.x version that is greater than or equal to 3.1.0 but less than 4.0.0.
Example Use Case:
Triggering a resource with no actual action:
``` shell
resource "null_resource" "example" {
  triggers = {
    always_run = "${timestamp()}"
  }
}
```
- 2.6 CloudInit Provider
``` shell
cloudinit = {
  source  = "hashicorp/cloudinit"
  version = "~> 2.2.0"
}
```
Purpose: The cloudinit provider is used to configure instances with cloud-init, a script that automates tasks such as setting up SSH keys, installing packages, or running commands.
Version Constraint:
~> 2.2.0 means:
Use any 2.x version that is greater than or equal to 2.2.0 but less than 3.0.0.
Example Use Case:
Provisioning an EC2 instance with a cloud-init script:
``` shell
resource "cloudinit_config" "example" {
  part {
    content_type = "text/cloud-config"
    content      = "packages:\n  - nginx\n"
  }
}
```

### ***Step 4: Explaining Terraform "variables.tf" configuration***
The variables.tf file defines input variables for the Terraform configuration. These variables make the code more reusable, flexible, and easier to manage across different environments.

#### 1. variable "kubernetes_version"
``` shell
variable "kubernetes_version" {
  default     = 1.27
  description = "Kubernetes version"
}
```
- Purpose: Specifies the version of Kubernetes to be used for the EKS (Elastic Kubernetes Service) cluster.
- default = 1.27: Sets the default Kubernetes version to 1.27.
Description: Provides a brief description of the variable.
#### 2. variable "vpc_cidr"
``` shell
variable "vpc_cidr" {
  default     = "10.0.0.0/16"
  description = "Default CIDR range of the VPC"
}
```
- Purpose: Defines the IP address range (CIDR block) for the VPC (Virtual Private Cloud).
- default = "10.0.0.0/16": Sets the default CIDR block to 10.0.0.0/16, which provides up to 65,536 IP addresses for the VPC.
Description: Indicates that this is the default IP range for the VPC.
#### 3. variable "aws_region"
``` shell
variable "aws_region" {
  default     = "us-west-1"
  description = "AWS region"
}
```
- Purpose: Specifies the AWS region where the infrastructure will be deployed.
- default = "us-west-1": Sets the default region to us-west-1 (Northern California).
Description: Provides a brief description of the variable.



### ***Step 4: Explaining Terraform "security-groups.tf" configuration***
The security-groups.tf file defines an AWS Security Group and its associated rules for managing network access to worker nodes in an Amazon EKS (Elastic Kubernetes Service) cluster.

#### 1. Security Group: aws_security_group
``` shell 
resource "aws_security_group" "all_worker_mgmt" {
  name_prefix = "all_worker_management"
  vpc_id      = module.vpc.vpc_id
}
```
- Purpose: Creates a security group named with a prefix all_worker_management for the EKS worker nodes.
- name_prefix: Appends a random suffix to ensure the name is unique.
- vpc_id = module.vpc.vpc_id: Associates the security group with the VPC created by the vpc module.
#### 2. Ingress Rule: aws_security_group_rule (Inbound Traffic)
``` shell  
resource "aws_security_group_rule" "all_worker_mgmt_ingress" {
  description       = "allow inbound traffic from eks"
  from_port         = 0
  protocol          = "-1"
  to_port           = 0
  security_group_id = aws_security_group.all_worker_mgmt.id
  type              = "ingress"
  cidr_blocks = [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
  ]
}
```
Purpose: Allows inbound (ingress) traffic to the worker nodes from private IP address ranges commonly used in AWS and on-premises networks.
description: Provides a brief description of the rule.
- from_port = 0 and to_port = 0: Allows all ports (0 is a wildcard value).
- protocol = "-1": Allows all protocols (e.g., TCP, UDP, ICMP).
- security_group_id: Associates the rule with the all_worker_mgmt security group.
- type = "ingress": Specifies that this is an inbound rule.
- cidr_blocks: Allows traffic from the following private IP ranges:
  10.0.0.0/8: Private IP range often used in AWS.
  172.16.0.0/12: Another private IP range.
  192.168.0.0/16: Common private IP range used in local networks.
  
#### 3. Egress Rule: aws_security_group_rule (Outbound Traffic)
``` shell
resource "aws_security_group_rule" "all_worker_mgmt_egress" {
  description       = "allow outbound traffic to anywhere"
  from_port         = 0
  protocol          = "-1"
  security_group_id = aws_security_group.all_worker_mgmt.id
  to_port           = 0
  type              = "egress"
  cidr_blocks       = ["0.0.0.0/0"]
}
```
Purpose: Allows outbound (egress) traffic from the worker nodes to anywhere on the internet.
description: Provides a brief description of the rule.
- from_port = 0 and to_port = 0: Allows all ports.
- protocol = "-1": Allows all protocols.
- security_group_id: Associates the rule with the all_worker_mgmt security group.
- type = "egress": Specifies that this is an outbound rule.
- cidr_blocks = ["0.0.0.0/0"]: Allows traffic to any destination on the internet.


### ***Step 5: Explaining Terraform "eks-cluster.tf" configuration***
This file defines the creation of an Amazon EKS (Elastic Kubernetes Service) cluster and its managed node groups using the Terraform AWS EKS module.

#### 1. module "eks": Creating the EKS Cluster
``` shell
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  version         = "20.8.4"
  cluster_name    = local.cluster_name
  cluster_version = var.kubernetes_version
  subnet_ids      = module.vpc.private_subnets
  enable_irsa     = true

  tags = {
    cluster = "demo"
  }

  vpc_id = module.vpc.vpc_id
}
```
- cluster_name: Dynamic name for the cluster.
- cluster_version: Kubernetes version (1.27).
- subnet_ids: Uses private subnets from the VPC.
- enable_irsa: Enables IAM Roles for Service Accounts (IRSA).
#### 2. eks_managed_node_group_defaults: Node Group Defaults
``` shell 
eks_managed_node_group_defaults = {
  ami_type               = "AL2_x86_64"
  instance_types         = ["t3.medium"]
  vpc_security_group_ids = [aws_security_group.all_worker_mgmt.id]
}
```
- ami_type: Amazon Linux 2 AMI.
- instance_types: Worker nodes are t3.medium.
- vpc_security_group_ids: Links to the security group.
#### 3. eks_managed_node_groups: Node Group Configuration
``` shell
eks_managed_node_groups = {
  node_group = {
    min_size     = 2
    max_size     = 6
    desired_size = 2
  }
}
```
Autoscaling:
- Minimum nodes: 2
- Maximum nodes: 6
- Initial nodes: 2


### ***Step 6: Explaining Terraform "outputs.tf" configuration***

This file defines output variables to display key information about the EKS cluster and its associated resources. These outputs make it easier to access and manage the infrastructure after Terraform has completed the deployment.

#### 1. EKS Cluster ID
``` shell
output "cluster_id" {
  description = "EKS cluster ID."
  value       = module.eks.cluster_id
}
```
- Purpose: Returns the unique identifier of the EKS cluster, used for managing or referencing the cluster in other AWS services.

#### 2. EKS Cluster Endpoint
``` shell
output "cluster_endpoint" {
  description = "Endpoint for EKS control plane."
  value       = module.eks.cluster_endpoint
}
```
- Purpose: Provides the URL endpoint for the EKS control plane, which is required to interact with the Kubernetes API.
#### 3. Cluster Security Group ID
``` shell
output "cluster_security_group_id" {
  description = "Security group IDs attached to the cluster control plane."
  value       = module.eks.cluster_security_group_id
}
```
- Purpose: Displays the security group ID used to manage network access to the cluster's control plane.
#### 4. AWS Region
``` shell
output "region" {
  description = "AWS region"
  value       = var.aws_region
}
```
- Purpose: Returns the AWS region where the EKS cluster is deployed.
#### 5. OIDC Provider ARN
``` shell
output "oidc_provider_arn" {
  value = module.eks.oidc_provider_arn
}
```
- Purpose: Outputs the ARN of the OIDC provider used for enabling IAM roles for Kubernetes service accounts (IRSA).

### ***Step 6: Run Terraform Commands***
Throughout the project, I followed Terraform best practices:
-  ***terraform init*** to initializes the working directory and downloads the necessary provider plugins.

![Screenshot 2024-12-03 141354](https://github.com/user-attachments/assets/acc578c0-9f3f-4bbd-9806-349c08899367)


- Used the ***terraform validate*** command to check for syntax errors.
- Run ***terraform fmt*** to format the code for readability.
- And executed ***terraform plan*** to preview the resources before applying changes.

![Screenshot 2024-12-03 141654](https://github.com/user-attachments/assets/e912f2e3-9c82-42ee-b0e8-dc96d8e28cb2)

- Finally, ***terraform apply*** to actually execute resources:
  
![Screenshot 2024-12-03 141827](https://github.com/user-attachments/assets/ec6ee525-541a-4bfb-8751-8652eb2c7c16)
![Screenshot 2024-12-03 143514](https://github.com/user-attachments/assets/368efa52-e55c-4634-997f-9d7723511ae2)

### Outputs:
![Screenshot 2024-12-03 143554](https://github.com/user-attachments/assets/35f3c538-70da-48bb-b28e-168c1a2e7b7b)
![Screenshot 2024-12-03 143831](https://github.com/user-attachments/assets/19ac6cab-8ed0-4d7f-a6d9-e39dd646992b)
![Screenshot 2024-12-03 143905](https://github.com/user-attachments/assets/d88abbdb-c938-4dd4-add2-27650206312a)
![Screenshot 2024-12-03 143831](https://github.com/user-attachments/assets/601527d6-0ce3-4f18-8cdd-3d9888d05c03)

- To View Nodes, we need to add Policy :
![Screenshot 2024-12-03 143636](https://github.com/user-attachments/assets/8bdc1ba6-c0c6-4996-858d-853f5e0db982)
![Screenshot 2024-12-03 143704](https://github.com/user-attachments/assets/f8fafc00-07e3-4aeb-967b-fdd001ca0b14)
Click Add Policy, then "Next"
![Screenshot 2024-12-03 143944](https://github.com/user-attachments/assets/fa36521d-03d2-431a-bc21-800ef64f4d04)
![Screenshot 2024-12-03 144752](https://github.com/user-attachments/assets/5c5f12b8-8e9c-4b7f-856d-3702484671a7)

#### Other Outputs :
![Screenshot 2024-12-03 144239](https://github.com/user-attachments/assets/e936e0ae-d9d3-4ec7-bec8-ef257d43f2c3)
![Screenshot 2024-12-03 144256](https://github.com/user-attachments/assets/927b2dca-89bb-45bf-9dbb-d2f1e9a060f1)

![Screenshot 2024-12-03 144158](https://github.com/user-attachments/assets/60643d82-d0eb-421c-ba4f-ee39d0c41260)
![Screenshot 2024-12-03 144326](https://github.com/user-attachments/assets/6daa7d19-62c4-4a5a-a962-0bc76af98dd6)
![Screenshot 2024-12-03 144158](https://github.com/user-attachments/assets/19ac8cdc-51a3-40a3-96b3-9d655a2a5256)
![Screenshot 2024-12-03 144031](https://github.com/user-attachments/assets/a628f22d-3907-40c3-882f-7aab9012c483)

