# Deploying an AWS VPC, Public and Private Subnets with NAT using Terraform.

![](https://miro.medium.com/max/1400/0*TBBrFvlIpywPioWT.png)

This article uses Terraform code to deploy AWS VPC, Public and Private Subnets with NAT Gateway.

You can utilize the code explained here to build your own infra as I have tried to keep it generic in terms of the number of Subnets and Availability Zones. This number is decided by the selection of your desired region.

**Prerequisites for Using Terraform to Create Infrastructure on AWS**

For creating and deleting permissions for all AWS resources, we require AWS IAM API keys (access key and secret key). The machine should have Terraform installed.

You have to create a folder and initialize Terraform using the following command.
```
$ terraform init
```
![](https://miro.medium.com/max/1400/1*MJfTvy0VCIRU2BuoKATzUg.png)

### Amazon Resources Created Using Terraform ###

1.  AWS VPC with required CIDR. You can change the default value in the variable “cidr_vpc”.
```
variable "cidr_vpc" {  
  default = "172.16.0.0/16"  
}
````
2. Multiple AWS VPC public subnets would be accessible via the internet.

3. Multiple AWS VPC private subnets would be inaccessible via the internet without the use of a NAT Gateway.

4. Deploy AWS VPC Internet Gateway and attach it to AWS VPC.

5. AWS VPC Route Tables, both public and private.

6. AWS VPC NAT Gateway attached with an Elastic Public IP.

7. Associating AWS VPC Subnets with VPC route tables.

## Step 1 . Define and configure Terraform providers. ##

The provider.tf file tells Terraform which provider to use. Because of the provider “aws,” all infrastructure will be hosted on AWS. Here we are defining the region and authentication required for the IAM user, which is used by Terraform to do AWS operations.
```
/*==== Provider ======*/  
/* Setting up of provider name and associated authentication */  
  
provider "aws" {  
  
  region     = var.region  
  access_key = var.access_key  
  secret_key = var.secret_key  
  
  default_tags {  
    tags = local.common_tags  
  }  
}
```
Default tags will be attached to all the resources created by terraform.
```
locals {  
  common_tags = {  
    project     = var.project  
    environment = var.environment  
    owner       = var.owner  
    application = var.application  
  }  
}
```
**Step 2: Variables are defined inside variables.tf file.**
```
/*==== Variable declerations ======*/  
  
variable "project" {  
  
  default     = "swiggy"  
  description = "Name of the project"  
}  
  
variable "environment" {  
  
  default     = "production"  
  description = "Name of the environment"  
}  
  
variable "region" {  
  
  default     = "ap-south-1"  
  description = "Region: Mumbai"  
}  
  
variable "access_key" {  
  
  default     = "AKIAVMDQREWV2RQ62KVQ"  
  description = "access key of the provider"  
}  
  
variable "secret_key" {  
  
  default     = "xjZYNrNuF6dIjAhWu2iwqG3cN2btCikkQod2zSb6"  
  description = "secret key of the provider"  
}  
  
variable "instance_ami" {  
  
  default     = "ami-0cca134ec43cf708f"  
  description = "Amazon Linux AMI"  
}  
  
variable "instance_type" {  
  
  default = "t2.micro"  
}  
  
variable "owner" {  
  
  default = "pratheesh"  
}  
  
variable "application" {  
  
  default = "food-order"  
}  
  
locals {  
  common_tags = {  
    project     = var.project  
    environment = var.environment  
    owner       = var.owner  
    application = var.application  
  }  
}  
  
variable "cidr_vpc" {  
  default = "172.16.0.0/16"  
}  
  
locals {  
  subnets = length(data.aws_availability_zones.available_azs.names)  
}
```
**Step 3: Datasource.tf file to gather information of already existing infrastructure in aws.**

It is essential to gather the data of availability zones in the region declared in the var.region variable. This will help to make this code to work anywhere irrespective of your intended region.
```
/*==== Gatthering of availability zones in the present region from datasource ======*/  
  
data "aws_availability_zones" "available_azs" {  
  state = "available"  
}
```
We can get the number of the availability zones using the following code, which can be utilized to decide the number of public and private subnets.
```
#variables.tf  
locals {  
  subnets = length(data.aws_availability_zones.available_azs.names)  
}
```
**Step 4: Creation of AWS Virtual Private Cloud.**

vpc cidr is defined in the variable **cidr_vpc.** You have to enable dns hostnames and if you require a dedicated instance you can keep instance_tenancy **true.**
```
/*==== vpc ======*/  
/*create vpc in the cidr "172.16.0.0/16" */  
  
resource "aws_vpc" "vpc" {  
  cidr_block           = var.cidr_vpc  
  enable_dns_hostnames = true  
  enable_dns_support   = true  
  instance_tenancy     = "default"  
  tags = {  
    Name = "${var.project}-${var.environment}"  
  }  
}
```
**Step 5: Creation of Internet Gateway for the VPC**

Public access from and to the VPC is possible only through Internet Gateway. Here we are creating and attaching the Internet Gateway.
```
/*==== IGW ======*/  
/* Create internet gateway for the public subnets and attach with vpc */  
  
resource "aws_internet_gateway" "igw" {  
  vpc_id = aws_vpc.vpc.id  
  
  tags = {  
    Name = "${var.project}-${var.environment}"  
  }  
}
```
**Step 6: Creation of Public Subnets.**

As we already now the number of AZs and so the number of subnets can be one per Availability Zone (Its purely a choice of the situation.) You can assign the **count meta argument** with the number of required subnets. Selection of subnet through cidrsubnet() function gives us flexibility to select the required subnet block and here we are using 4 bits (/20) for subnetting. 16 (2⁴) subnets will be available for our use and each of them consists of 4094 host addresses that is very convenient.
```
/*==== Public Subnets ======*/  
/* Creation of Public subnets, one for each availability zone in the region  */  
  
resource "aws_subnet" "public" {  
  count = local.subnets  
  vpc_id                  = aws_vpc.vpc.id  
  cidr_block              = cidrsubnet(var.cidr_vpc, 4, count.index)  
  availability_zone       = data.aws_availability_zones.available_azs.names[count.index]  
  map_public_ip_on_launch = true  
  tags = {  
    Name = "${var.project}-${var.environment}-public${count.index + 1}"  
  }  
}
```
As the default region here is “ap-south-1” we will have 3 Availability Zones and so 3 public and private subnets each.

Selection of AZ while creating subnet is desided by the following line of code.

availability_zone       = data.aws_availability_zones.available_azs.names[count.index]

You have to keep the **map_public_ip_on_launch = true** for the public accesses of the instance in the public subnet.

**Step 7: Creation of Private Subnets.**

You have to keep the **map_public_ip_on_launch = fa**lse for restricting default public IP assignment of the instances.
```
/*==== Private Subnets ======*/  
/* Creation of Private  subnets, one for each availability zone in the region  */  
  
resource "aws_subnet" "private" {  
  count = local.subnets  
  vpc_id                  = aws_vpc.vpc.id  
  cidr_block              = cidrsubnet(var.cidr_vpc, 4, (count.index + local.subnets))  
  availability_zone       = data.aws_availability_zones.available_azs.names[count.index]  
  map_public_ip_on_launch = false  
  tags = {  
    Name = "${var.project}-${var.environment}-private${count.index + 1}"  
  }  
}
```
**Step 8: Creation of Elastic IP and Attachment with NAT Gateway.**

NAT Gateway is a device which helps the instances in the private subnet to access the internet. It requires an Elastic IP to do the job. Placement of the NAT GW should be in any of the public subnets.
```
/*==== Elastic IP ======*/  
/* Creation of Elastic IP for  NAT Gateway */  
  
resource "aws_eip" "nat_ip" {  
  vpc = true  
}  
  
/*==== Creation of NAT Gateway Elastic IP Attachment ======*/  
/* Creation of NAT Gateway in the second public subnet and  
 Attachment of Elastic IP for the public access of NAT Gateway */  
  
resource "aws_nat_gateway" "nat_gw" {  
  allocation_id = aws_eip.nat_ip.id  
  subnet_id     = aws_subnet.public[2].id  
  
  tags = {  
    Name = "${var.project}-${var.environment}"  
  }  
  
  # To ensure proper ordering, it is recommended to add an explicit dependency  
  # on the Internet Gateway for the VPC.  
  depends_on = [aws_internet_gateway.igw]  
}
```
**Step 9a: Creation of route for public access via the Internet gateway for the vpc.**

We have to create the public route table and add a route through the IGW to the public IP space.
```
/*==== Public Route Table ======*/  
  
resource "aws_route_table" "public" {  
  vpc_id = aws_vpc.vpc.id  
  
  route {  
    cidr_block = "0.0.0.0/0"  
    gateway_id = aws_internet_gateway.igw.id  
  }  
  
  tags = {  
    Name = "${var.project}-${var.environment}-public"  
  }  
}
```
**Step 9b: Association of Public Route Table with all the public subnets is a requirement for their public access.**

Selection of the subnet is done using count meta argument.
```
/*==== Association Public Route Table ======*/  
/*Association of Public route table with public subnets. */  
  
resource "aws_route_table_association" "public" {  
  count = local.subnets  
  subnet_id      = aws_subnet.public[count.index].id  
  route_table_id = aws_route_table.public.id  
}
```
**Step 10a: Creation of Private Route Table with route for public access via the NAT gateway.**

**nat_gateway_id** is a mandatory requirement for this step, which requires the creation of NAT GW.
```
/*==== Private Route Table ======*/  
/*Creation of Private Route Table with route for public access via the NAT gateway */  
  
resource "aws_route_table" "private" {  
  vpc_id = aws_vpc.vpc.id  
  
  route {  
    cidr_block     = "0.0.0.0/0"  
    nat_gateway_id = aws_nat_gateway.nat_gw.id  
  }  
  
  tags = {  
    Name = "${var.project}-${var.environment}-private"  
  }  
}
```
**Step 10b : Association of Private Route Table with all the private subnets is a requirement.**

Selection of the subnet is done using **count meta argument.**
```
/*==== Association Private Route Table ======*/  
/*Association of Private route table with private subnets. */  
  
resource "aws_route_table_association" "private" {  
  count = local.subnets  
  subnet_id      = aws_subnet.private[count.index].id  
  route_table_id = aws_route_table.private.id  
}
```
**Step 11 : After coding we have to Plan the code to build the infrastructure.**

Let’s run `terraform fmt` to format our code and make it beautiful.Then, run`terraform validate`to validate the files.

**terraform plan** command will provide us an overview about the infrastructural changes that are going to happen if we allow Terraform to do. The following output is edited to give a concise view.
```
$ terraform plan  
 Reading...  
Read complete after 1s [id=ap-south-1]  
  
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated  
with the following symbols:  
  + create  
  
Terraform will perform the following actions:  
  
  # aws_eip.nat_ip will be created  
 ....  
  
  # aws_internet_gateway.igw will be created  
 ....  
  # aws_nat_gateway.nat_gw will be created  
 ....  
  
  # aws_route_table.private will be created  
 ....  
  
  # aws_route_table.public will be created  
 ....  
    }  
  
  # aws_route_table_association.private[0] will be created  
....  
  # aws_route_table_association.private[1] will be created  
....  
  
  # aws_route_table_association.private[2] will be created  
....  
  
  # aws_route_table_association.public[0] will be created  
....  
  
  # aws_route_table_association.public[1] will be created  
....  
  # aws_route_table_association.public[2] will be created  
....  
  
  # aws_subnet.private[0] will be created  
....  
  
  # aws_subnet.private[1] will be created  
....  
  
  # aws_subnet.private[2] will be created  
....  
  
  # aws_subnet.public[0] will be created  
....  
  # aws_subnet.public[1] will be created  
....  
  # aws_subnet.public[2] will be created  
....  
  # aws_vpc.vpc will be created  
  + resource "aws_vpc" "vpc" {  
      + arn                                  = (known after apply)  
      + cidr_block                           = "172.16.0.0/16"  
      + default_network_acl_id               = (known after apply)  
      + default_route_table_id               = (known after apply)  
      + default_security_group_id            = (known after apply)  
      + dhcp_options_id                      = (known after apply)  
      + enable_classiclink                   = (known after apply)  
      + enable_classiclink_dns_support       = (known after apply)  
      + enable_dns_hostnames                 = true  
      + enable_dns_support                   = true  
      + enable_network_address_usage_metrics = (known after apply)  
      + id                                   = (known after apply)  
      + instance_tenancy                     = "default"  
      + ipv6_association_id                  = (known after apply)  
      + ipv6_cidr_block                      = (known after apply)  
      + ipv6_cidr_block_network_border_group = (known after apply)  
      + main_route_table_id                  = (known after apply)  
      + owner_id                             = (known after apply)  
      + tags                                 = {  
          + "Name" = "swiggy-production"  
        }  
      + tags_all                             = {  
          + "Name"        = "swiggy-production"  
          + "application" = "food-order"  
          + "environment" = "production"  
          + "owner"       = "pratheesh"  
          + "project"     = "swiggy"  
        }  
    }  
  
Plan: 18 to add, 0 to change, 0 to destroy.
```
**Step 12 : terraform apply command to implement the aws infrastructure.**

The terraform apply is a command is used to apply the changes required to reach the desired state of the configuration from the present state, or the pre-determined set of actions generated by a terraform plan execution plan.
```
$ terraform apply -auto-approve  
.  
.  
<planning phase>  
.  
.  
aws_vpc.vpc: Creating...  
aws_eip.nat_ip: Creating...  
aws_eip.nat_ip: Creation complete after 1s [id=eipalloc-034ef8d818e92111d]  
aws_vpc.vpc: Still creating... [10s elapsed]  
aws_vpc.vpc: Creation complete after 13s [id=vpc-04a70ed160dfc4f65]  
aws_subnet.public[0]: Creating...  
aws_internet_gateway.igw: Creating...  
aws_subnet.private[2]: Creating...  
aws_subnet.private[0]: Creating...  
aws_subnet.private[1]: Creating...  
aws_subnet.public[2]: Creating...  
aws_subnet.public[1]: Creating...  
aws_internet_gateway.igw: Creation complete after 2s [id=igw-049b7aae26a4662fe]  
aws_route_table.public: Creating...  
aws_route_table.public: Creation complete after 1s [id=rtb-041c9cba51b679e9f]  
aws_subnet.public[0]: Still creating... [10s elapsed]  
aws_subnet.public[2]: Still creating... [10s elapsed]  
aws_subnet.private[1]: Still creating... [10s elapsed]  
aws_subnet.private[2]: Still creating... [10s elapsed]  
aws_subnet.public[1]: Still creating... [10s elapsed]  
aws_subnet.private[0]: Still creating... [10s elapsed]  
aws_subnet.public[0]: Creation complete after 12s [id=subnet-0cb8c392718f68a1e]  
aws_subnet.private[2]: Creation complete after 12s [id=subnet-04c43a2eb731df2b7]  
aws_subnet.private[1]: Creation complete after 12s [id=subnet-0dce85a1f0ac06402]  
aws_subnet.private[0]: Creation complete after 12s [id=subnet-03ddb8398f791fa27]  
aws_subnet.public[1]: Creation complete after 12s [id=subnet-0bcd303e1accb5755]  
aws_subnet.public[2]: Creation complete after 12s [id=subnet-0d8c833d396ba2d44]  
aws_route_table_association.public[0]: Creating...  
aws_nat_gateway.nat_gw: Creating...  
aws_route_table_association.public[2]: Creating...  
aws_route_table_association.public[1]: Creating...  
aws_route_table_association.public[1]: Creation complete after 1s [id=rtbassoc-0628bbd3b4b74c239]  
aws_route_table_association.public[0]: Creation complete after 1s [id=rtbassoc-02e2fe6fba12283b0]  
aws_route_table_association.public[2]: Creation complete after 1s [id=rtbassoc-07722bf085edb9c19]  
aws_nat_gateway.nat_gw: Still creating... [10s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [20s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [30s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [40s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [50s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [1m0s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [1m10s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [1m20s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [1m30s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [1m40s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [1m50s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [2m0s elapsed]  
aws_nat_gateway.nat_gw: Still creating... [2m10s elapsed]  
aws_nat_gateway.nat_gw: Creation complete after 2m17s [id=nat-008663d7e8213693b]  
aws_route_table.private: Creating...  
aws_route_table.private: Creation complete after 2s [id=rtb-0d5a5e64d227f0eba]  
aws_route_table_association.private[1]: Creating...  
aws_route_table_association.private[0]: Creating...  
aws_route_table_association.private[2]: Creating...  
aws_route_table_association.private[1]: Creation complete after 1s [id=rtbassoc-04ad84b565593ffa5]  
aws_route_table_association.private[2]: Creation complete after 1s [id=rtbassoc-01ee30ddcf2efb93e]  
aws_route_table_association.private[0]: Creation complete after 1s [id=rtbassoc-0ec7970aa06d4557d]  
  
Apply complete! Resources: 18 added, 0 changed, 0 destroyed.
```
**# created vpc.**

![](https://miro.medium.com/max/1400/1*eQTizhvY8l_ELoajcFbcuw.png)

**#created subnets.**

![](https://miro.medium.com/max/1400/1*GCcJOb8QVLos-e9ET3Iitg.png)

**#Subnets’ association Route Tables**

![](https://miro.medium.com/max/1400/1*WeOnWqgUukI942Yten9LiA.png)
Incase of any suggestions please contact me @ yespratheesh@gmail.com 
Thank you for your time.

**Read my articles in medium.com.**

  <a target="_blank" href="https://github-readme-medium-recent-article.vercel.app/medium/@yespratheesh/2"><img src="https://github-readme-medium-recent-article.vercel.app/medium/@yespratheesh/2">
  
 <a target="_blank" href="https://github-readme-medium-recent-article.vercel.app/medium/@yespratheesh/1"><img src="https://github-readme-medium-recent-article.vercel.app/medium/@yespratheesh/1">
