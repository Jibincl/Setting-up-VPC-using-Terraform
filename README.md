# Setting-up-VPC-using-Terraform
Amazon Virtual Private Cloud (Amazon VPC) enables you to launch Amazon Web Services resources into a virtual network you've defined. This virtual network resembles a traditional network that you'd operate in your own data center, with the benefits of using the scalable infrastructure of AWS. Here we are going to discuss on creating a complete VPS setup using terraform. 

### Prerequisites
1. IAM user with administrator access to EC2.
2. Terraform is to be installed on the instance

Lets get into this.

Lets create a file for declaring the variables.This is used to declare the variable and the values are passing through the terrafrom.tfvars file.

### Create a variables.tf file

~~~
variable "region" {}
variable "access_key" {}
variable "secret_key" {}
variable "vpc_cidr" {}
variable "project" {}
~~~

### Creating the provider.tf file

This files contains the provider configuration.

~~~
provider "aws" {
  region     = var.region
  access_key = var.access_key
  secret_key = var.secret_key
}
~~~

### Create a terraform.tfvars
By default terraform.tfvars will load the variables to the the resources. You can modify accordingly as per your requirements.

~~~
region = "put-your-region-here"
access_key = "put-your-access_key-here"
secret_key = "put-your-secret_key-here"
vpc_cidr = "X.X.X.X/16"
project = "name-of-your-project"
~~~

Enter the command given below to initialize a working directory containing Terraform configuration files. This is the first command that should be run after writing a new Terraform configuration.

~~~
terraform init
~~~
Now a terraform. tfstate file would be generated here.

### Creating the main .tf file
The main configuration file has the following contents

### To create VPC

~~~
resource "aws_vpc" "main" {

   cidr_block = var.vpc_cidr
   
   instance_tenancy = "default"

    enable_dns_support = true

        enable_dns_hostnames = true

    tags = {
        Name = var.project
    }

}
~~~

### To Gather All Subnet Name

~~~
data "aws_availability_zones" "available" {
  state = "available"
}
~~~

### To create InterGateWay For VPC

~~~
resource "aws_internet_gateway" "igw" {

  vpc_id = aws_vpc.main.id

  tags = {
    Name = var.project
  }
}
~~~

Here in this infrastructre we shall create 3 public and 3 private subnets in the region.This sample was meant for regions having 6 availability zone. I have used "us-east-1". Choose your region and modify according to the availability of the AZ. Also we have already provided the CIDR block in our terraform.tfvars you dont need to calculate the subnets, here we use terraform to automate the subnetting in /19.

### Creating public1, public2, public3 Subnet using count

~~~
resource "aws_subnet" "public" {

  count = 3
 
  vpc_id     = aws_vpc.main.id

  cidr_block = cidrsubnet(var.vpc_cidr, 3, count.index)

  availability_zone = data.aws_availability_zones.subnet.names[count.index]

  map_public_ip_on_launch = true


tags = {
    Name = "${var.project}-public${count.index+1}"

  }
}
~~~

### Creating private1, private2, private3 Subnet using count

~~~
resource "aws_subnet" "private" {

  count = 3
 
  vpc_id     = aws_vpc.main.id

  cidr_block = cidrsubnet(var.vpc_cidr, 3, "${count.index+3}")

  availability_zone = data.aws_availability_zones.subnet.names[count.index]

  map_public_ip_on_launch = false


tags = {
    Name = "${var.project}-private${count.index+1}"

  }
}
~~~

### Creating Elastic IP For Nat Gateway

~~~
resource "aws_eip" "eip" {
  vpc      = true
  tags     = {
    Name = "${var.project}-eip"
  }
}
~~~

### Attaching Elastic IP to NAT gateway

~~~
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.eip.id
  subnet_id     = aws_subnet.public1.id
  tags = {
    Name = "${var.project}-nat"
  }
}
~~~

###  Creating Public Route Table

~~~
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

   route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

   tags = {
    Name = "${var.project}-public"
  }
}
~~~

### Creating Private Route Table

~~~
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }

   tags = {
    Name = "${var.project}-private"
  }
}
~~~

### Creating Public Route Table Association

~~~
resource "aws_route_table_association" "public" {

  count =3 
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id

  depends_on = [
    aws_subnet.public ]
}
~~~

### Creating Private Route Table Association

~~~
resource "aws_route_table_association" "private" {

  count =3 
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id

  depends_on = [
    aws_subnet.private ]
}
~~~

### Create an output.tf for getting terrafrom output.

~~~
output "aws_eip" {
value = aws_eip.eip.public_ip
}
output "aws_vpc" {
value = aws_vpc.main.id
}
output "aws_internet_gateway" {
value = aws_internet_gateway.igw.id
}
output "aws_nat_gateway" {
value = aws_nat_gateway.nat.id
}
output "aws_route_table_public" {
value = aws_route_table.public.id
}
output "aws_route_table_private" {
value = aws_route_table.private.id
}
~~~

### Now, inorder to validate the terraform files, run the following command

~~~
terraform validate
~~~

### Now, inorder to create and verify the execution plan, run the following command:

~~~
terraform plan
~~~

### Now, let us executes the actions proposed in a Terraform plan by using the following command

~~~
terraform plan
~~~

###  Now, let us executes the actions proposed in a Terraform plan by using the following command

~~~
terraform apply
~~~

### Conclusion

Here, we created a complete VPC using terraform. 
