---
layout: post
title:  "Terraform管理AWS基础架构之一"
date:   2017-09-02 12:28:53 +0800
categories: aws terraform
---

# Terraform管理AWS基础架构之一

Infrastructure as Code(基础设施即代码)，可以很方便地管理我们的基础设施，这是我的一个测试案例：

本案例使用Terraform管理AWS基础架构:
* 一个VPC
* 两个子网
* Internet Gateway
* 默认安全组
* 默认路由表
* 两个EC2实例，分布在两个不同的Availability Zone
* 两个EIP
* 一个负载均衡器，监听80端口


创建myaws.tf文件，内容如下：

你需要至少修改三处地方
* ACCESS_KEY
* SECRET_KEY
* ssh公钥

```
provider "aws" {
  access_key = "YOU ACCESS KEY"
  secret_key = "YOU SECRET KEY XXX XXX XXX"
  region     = "us-east-1"
}


resource "aws_vpc" "terraform" {
  cidr_block       = "192.168.168.0/24"

  tags {
    Name = "terraform.tag"
  }
}


resource "aws_subnet" "terraform_subnet" {
  vpc_id     = "${aws_vpc.terraform.id}"
  cidr_block = "192.168.168.128/25"
  availability_zone = "us-east-1b"

  tags {
    Name = "terraform"
  }
}


resource "aws_subnet" "terraform_subnet02" {
  vpc_id     = "${aws_vpc.terraform.id}"
  cidr_block = "192.168.168.0/25"
  availability_zone = "us-east-1c"

  tags {
    Name = "terraform02"
  }
}


resource "aws_internet_gateway" "gw" {
  vpc_id = "${aws_vpc.terraform.id}"
}


resource "aws_default_security_group" "default" {
  vpc_id = "${aws_vpc.terraform.id}"

  ingress {
    protocol  = -1
    self      = true
    from_port = 0
    to_port   = 0
  }

  ingress {
    protocol  = -1
    from_port = 0
    to_port   = 0
    cidr_blocks = ["0.0.0.0/0"]
  }


  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}


resource "aws_default_route_table" "r" {
  default_route_table_id = "${aws_vpc.terraform.default_route_table_id}"

  route {
	cidr_block = "0.0.0.0/0"
    	gateway_id = "${aws_internet_gateway.gw.id}"
  }

  tags {
    Name = "default table"
  }
}


resource "aws_key_pair" "terraform" {
  key_name   = "terraform-key"
  public_key = "ssh-rsa YOUR SSH PUBLIC KEY HERE XXX XXX XXX"
}


resource "aws_eip" "terraformEIP" {
  count    = 2
  instance = "${element(aws_instance.example.*.id, count.index)}"
  vpc      = true
}


resource "aws_instance" "example" {
  ami           = "ami-a4c7edb2"
  instance_type = "t2.micro"
  availability_zone = "${element(var.azs, count.index)}"
  key_name	= "terraform-key"
  count 	= "2"
  subnet_id 	= "${element(list("${aws_subnet.terraform_subnet.id}", "${aws_subnet.terraform_subnet02.id}"), count.index)}"
  depends_on 	= ["aws_internet_gateway.gw"]
  
  tags {
    Name = "terraform-${count.index}"
  }
}


variable "azs" {
  description = "Run the EC2 Instances in these Availability Zones"
  type = "list"
  default = ["us-east-1b", "us-east-1c"]
}


resource "aws_elb" "example" {
	name = "example"
#	availability_zones = ["us-east-1b","us-east-1c"]
	instances = ["${aws_instance.example.*.id}"]
	subnets = ["${aws_subnet.terraform_subnet.id}","${aws_subnet.terraform_subnet02.id}"]
	listener {
		lb_port = 80
		lb_protocol = "http"
		instance_port = "80"
		instance_protocol = "http"
	}
}

```



