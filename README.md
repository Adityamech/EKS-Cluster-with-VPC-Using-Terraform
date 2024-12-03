# Implement EKS Cluster with VPC Using Terraform for Real-time Scalable Infrastructure with Autoscaling Worker Nodes.


## ***Step 2 : Explaining Terraform Configuration Files***
#### 1. Explanation of vpc.tf File
The vpc.tf file defines the configuration to create a Virtual Private Cloud (VPC) in AWS using Terraform. This VPC will serve as the networking backbone for an EKS (Elastic Kubernetes Service) cluster.

1. AWS Provider Configuration
``` shell
provider "aws" {
  region = var.aws_region
}
```
Purpose: Configures Terraform to interact with AWS services.
region = var.aws_region: Specifies the AWS region where the resources will be created, using a variable (var.aws_region). This value must be defined in a variables.tf file or passed during execution.
2. Data Source: AWS Availability Zones
``` shell
data "aws_availability_zones" "available" {}
```
Purpose: Dynamically retrieves the list of available AWS Availability Zones (AZs) in the selected region.
Usage: Used later to distribute subnets across multiple AZs for high availability.
3. Local Variable: Cluster Name
``` shell
locals {
  cluster_name = "abhi-eks-${random_string.suffix.result}"
}
```
Purpose: Defines a local variable cluster_name to store the name of the EKS cluster.
random_string.suffix.result: Appends a random string to ensure the cluster name is unique.
4. Resource: Random String
``` shell
resource "random_string" "suffix" {
  length  = 8
  special = false
}
```
Purpose: Generates an 8-character alphanumeric random string to ensure the cluster name and resources remain unique.
special = false: Excludes special characters from the generated string.
5. Module: VPC (Virtual Private Cloud)
``` shell
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.7.0"
```
Source: Uses the official VPC module from the Terraform Registry (terraform-aws-modules/vpc/aws).
Version: The module version is locked to 5.7.0 to ensure compatibility and stability.
5.1 VPC Configuration
``` shell
  name                 = "abhi-eks-vpc"
  cidr                 = var.vpc_cidr
  azs                  = data.aws_availability_zones.available.names
```
name: Assigns a name to the VPC (abhi-eks-vpc).
cidr: The IP range for the VPC, specified by the variable var.vpc_cidr.
azs: Distributes subnets across all available availability zones.
5.2 Subnet Configuration
``` shell
  private_subnets      = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets       = ["10.0.4.0/24", "10.0.5.0/24"]
```
Private Subnets:
IP ranges for two private subnets (10.0.1.0/24 and 10.0.2.0/24). These are used for internal resources (e.g., worker nodes).

Public Subnets:
IP ranges for two public subnets (10.0.4.0/24 and 10.0.5.0/24). These are used for resources that need internet access (e.g., load balancers).

5.3 NAT Gateway Configuration
``` shell
  enable_nat_gateway   = true
  single_nat_gateway   = true
```
enable_nat_gateway: Enables a NAT Gateway for internet access from private subnets.
single_nat_gateway: Specifies the creation of a single NAT Gateway (for cost efficiency).
5.4 DNS Settings
``` shell
  enable_dns_hostnames = true
  enable_dns_support   = true
```
enable_dns_hostnames: Enables DNS hostnames for instances in the VPC.
enable_dns_support: Enables DNS resolution for the VPC.
5.5 Tags for Resources
``` shell
  tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
  }
```
Purpose: Tags the VPC to indicate it belongs to a Kubernetes cluster.
Key: "kubernetes.io/cluster/${local.cluster_name}"
Value: "shared" – Indicates that the VPC can be shared with Kubernetes resources.
5.6 Public Subnet Tags
``` shell
  public_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                      = "1"
  }
```
Purpose: Tags the public subnets for use with external load balancers.
kubernetes.io/role/elb = 1: Marks these subnets as eligible for external load balancers.
5.7 Private Subnet Tags
``` shell
  private_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"             = "1"
  }
```
Purpose: Tags the private subnets for use with internal load balancers.
kubernetes.io/role/internal-elb = 1: Marks these subnets as eligible for internal load balancers.

#### 2. Explanation of versions.tf File
The versions.tf file in Terraform is used to specify the version requirements for both Terraform and the providers it uses. It ensures compatibility between the infrastructure code, Terraform itself, and the providers that interact with external services like AWS or Kubernetes. Let’s break it down:

1. Terraform Block
The terraform block defines the Terraform version and required providers.

``` shell
terraform {
  required_version = ">= 0.12"
  required_providers {
    ...
  }
}
```
required_version = ">= 0.12"
Specifies that the Terraform version must be 0.12 or higher.
This ensures compatibility with modern Terraform syntax and features introduced after version 0.12.
2. Required Providers
The required_providers block specifies the providers required by the Terraform configuration, including their source and version constraints.

2.1 Random Provider
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
2.2 Kubernetes Provider
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
2.3 AWS Provider
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
2.4 Local Provider
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
2.5 Null Provider
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
2.6 CloudInit Provider
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

#### 3. Explanation of variables.tf File
The variables.tf file defines input variables for the Terraform configuration. These variables make the code more reusable, flexible, and easier to manage across different environments.

1. variable "kubernetes_version"
``` shell
variable "kubernetes_version" {
  default     = 1.27
  description = "Kubernetes version"
}
```
Purpose: Specifies the version of Kubernetes to be used for the EKS (Elastic Kubernetes Service) cluster.
default = 1.27: Sets the default Kubernetes version to 1.27.
Description: Provides a brief description of the variable.
2. variable "vpc_cidr"
``` shell
variable "vpc_cidr" {
  default     = "10.0.0.0/16"
  description = "Default CIDR range of the VPC"
}
```
Purpose: Defines the IP address range (CIDR block) for the VPC (Virtual Private Cloud).
default = "10.0.0.0/16": Sets the default CIDR block to 10.0.0.0/16, which provides up to 65,536 IP addresses for the VPC.
Description: Indicates that this is the default IP range for the VPC.
3. variable "aws_region"
``` shell
variable "aws_region" {
  default     = "us-west-1"
  description = "AWS region"
}
```
Purpose: Specifies the AWS region where the infrastructure will be deployed.
default = "us-west-1": Sets the default region to us-west-1 (Northern California).
Description: Provides a brief description of the variable.



#### 4. Explanation of security-groups.tf File
The security-groups.tf file defines an AWS Security Group and its associated rules for managing network access to worker nodes in an Amazon EKS (Elastic Kubernetes Service) cluster.

1. Security Group: aws_security_group
``` shell 
resource "aws_security_group" "all_worker_mgmt" {
  name_prefix = "all_worker_management"
  vpc_id      = module.vpc.vpc_id
}
```
Purpose: Creates a security group named with a prefix all_worker_management for the EKS worker nodes.
name_prefix: Appends a random suffix to ensure the name is unique.
vpc_id = module.vpc.vpc_id: Associates the security group with the VPC created by the vpc module.
2. Ingress Rule: aws_security_group_rule (Inbound Traffic)
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
from_port = 0 and to_port = 0: Allows all ports (0 is a wildcard value).
protocol = "-1": Allows all protocols (e.g., TCP, UDP, ICMP).
security_group_id: Associates the rule with the all_worker_mgmt security group.
type = "ingress": Specifies that this is an inbound rule.
cidr_blocks: Allows traffic from the following private IP ranges:
10.0.0.0/8: Private IP range often used in AWS.
172.16.0.0/12: Another private IP range.
192.168.0.0/16: Common private IP range used in local networks.
3. Egress Rule: aws_security_group_rule (Outbound Traffic)
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
from_port = 0 and to_port = 0: Allows all ports.
protocol = "-1": Allows all protocols.
security_group_id: Associates the rule with the all_worker_mgmt security group.
type = "egress": Specifies that this is an outbound rule.
cidr_blocks = ["0.0.0.0/0"]: Allows traffic to any destination on the internet.
