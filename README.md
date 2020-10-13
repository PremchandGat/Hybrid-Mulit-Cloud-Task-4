# Hybrid-Mulit-Cloud-Task-4
Performing the following steps:
1.  Write an Infrastructure as code using terraform, which automatically create a VPC.
2.  In that VPC we have to create 2 subnets:
    1.   public  subnet [ Accessible for Public World! ] 
    2.   private subnet [ Restricted for Public World! ]
3. Create a public facing internet gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC.
4. Create  a routing table for Internet gateway so that instance can connect to outside world, update and associate it with public subnet.
5.  Create a NAT gateway for connect our VPC/Network to the internet world  and attach this gateway to our VPC in the public network
6.  Update the routing table of the private subnet, so that to access the internet it uses the nat gateway created in the public subnet
7.  Launch an ec2 instance which has Wordpress setup already having the security group allowing  port 80 sothat our client can connect to our wordpress site. Also attach the key to instance for further login into it.
8.  Launch an ec2 instance which has MYSQL setup already with security group allowing  port 3306 in private subnet so that our wordpress vm can connect with the same. Also attach the key with the same.

Note: Wordpress instance has to be part of public subnet so that our client can connect our site. 
mysql instance has to be part of private  subnet so that outside world can't connect to it.
Don't forgot to add auto ip assign and auto dns name assignment option to be enabled.

Try each step first manually and write Terraform code for the same to have a proper understanding of workflow of task.
# How to use code
<pre>
1. First download terraform code
2. Do some changes in code according to requirement
   change aws profile name 
3. Run command <b>terraform init</b>
4. Run command <b>terraform apply</b> </pre>
# Create Terraform code
<pre>
provider "aws" {
  region  = "ap-south-1"
  profile = "prem" <b># Change profile name</b>
}
</pre>
# create a variable to store mysql password
<pre>
variable "mysql_password" {
  type = string
}
</pre>
# create a new vpc
<pre>
resource "aws_vpc" "main" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "new_vpc_terraform"
  }
}</pre>
# create private subnet
<pre>
resource "aws_subnet" "private" {
  depends_on = [aws_vpc.main,]
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "ap-south-1a"


  tags = {
    Name = "subnet_private"
  }
}</pre>
# create public subnet
<pre>
resource "aws_subnet" "public" {
  depends_on = [aws_subnet.private,]
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  map_public_ip_on_launch = true
  availability_zone = "ap-south-1b"
  tags = {
    Name = "subnet_public"
  }
}</pre>
# create a internat gateway
<pre>
resource "aws_internet_gateway" "gw" {
  depends_on = [aws_subnet.public,]
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "terraform_gateway"
  }
}</pre>
# create a routing table 
<pre>
resource "aws_route_table" "r" {
depends_on =   [aws_internet_gateway.gw,]
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
  tags = {
    Name = "route_terraform"
  }
}
</pre>
# associate routing tabel to public subnet
<pre>
resource "aws_route_table_association" "a" {
  depends_on = [aws_route_table.r,]
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.r.id
}
</pre>
# associate routing tabel to vpc
<pre>
resource "aws_main_route_table_association" "a" {
  depends_on = [aws_route_table_association.a,]
  vpc_id         = aws_vpc.main.id
  route_table_id = aws_route_table.r.id
}
</pre>
# creating a eip
<pre>
resource "aws_eip" "nat_gateway" {
  vpc = true
}
</pre>
# creatig a NAT gatway
<pre>
resource "aws_nat_gateway" "nat_gateway" {
  allocation_id = aws_eip.nat_gateway.id
  subnet_id = aws_subnet.private.id
  tags = {
    "Name" = "NatGateway"
  }
}</pre>
# create a routing table
<pre>
resource "aws_route_table" "instance" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat_gateway.id
  }
}
</pre>
# assoiate woth private subnet
<pre>
resource "aws_route_table_association" "instance" {
  subnet_id = aws_subnet.private.id
  route_table_id = aws_route_table.instance.id
}
</pre>
# create security group for wordpress instance
<pre>
resource "aws_security_group" "allow_wordpress" {
  depends_on =[aws_main_route_table_association.a,]
  name        = "security_created_by_terraform_for_wordpress"
  description = "Allow TLS inbound traffic"
  vpc_id      =  aws_vpc.main.id

  ingress {
    description = "TLS from VPC"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "TLS from VPC"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "terraform_wordpress_security_group"
  }
}
</pre>
# Create a security group for mysql instance
<pre>
resource "aws_security_group" "allow_mysql" {
  depends_on =[aws_main_route_table_association.a,]
  name        = "security_created_by_terraform_mysql"
  description = "Allow TLS inbound traffic"
  vpc_id      =  aws_vpc.main.id

  ingress {
    description = "TLS from VPC"
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "TLS from VPC"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "terraform_mysql_security_group"
  }
}
</pre>
# Launch mysql instance on aws
<pre>
resource "aws_instance" "mysql" {
   depends_on = [
  aws_security_group.allow_mysql ,
     ]
  ami           = "ami-0052c21533adedad3"
  instance_type = "t2.micro"
  key_name = "mykey"
  subnet_id = aws_subnet.private.id
  security_groups = [ aws_security_group.allow_mysql.id , ]
  user_data = <<-EOT
	  #! /bin/bash
    sudo systemctl start docker
    sudo docker run --name mysql1 -e MYSQL_ROOT_PASSWORD=${var.mysql_password} -d -p 3306:3306 mysql:5.7
	EOT
  tags = {
    Name = "mysql"
  }

}
</pre>
# Launch wordpress instance on aws
<pre>
resource "aws_instance" "wordpress" {
   depends_on = [
  aws_instance.mysql ,aws_security_group.allow_wordpress
     ]
  ami           = "ami-0447a12f28fddb066"
  instance_type = "t2.micro"
  key_name = "mykey"
  subnet_id = aws_subnet.public.id
  security_groups = [ aws_security_group.allow_wordpress.id , ]
  user_data = <<-EOT
		#! /bin/bash
    sudo yum install docker -y
    sudo systemctl start docker
    sudo docker container run -dit -e WORDPRESS_DB_HOST=${aws_instance.mysql.private_ip}:3306 -e WORDPRESS_DB_PASSWORD=${var.mysql_password} -e WORDPRESS_DB_USER=root -p 80:80  wordpress:php7.3
  EOT
  tags = {
    Name = "wordpress"
  }

}
</pre>
