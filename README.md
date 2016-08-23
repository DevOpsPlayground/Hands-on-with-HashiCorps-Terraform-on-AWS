# devopsplayground6-terraform


# Overview
This Devops Playground will explore Infrastructure as Code with Hashicorp Terraform and  AWS. The plan is to automate the creation of AWS resources, using terraform.  In order to display the full extent of Terraform's features, we will try to lauch two WebServer instances, and put them behind a load balancer, transparently, through a single, unified template.

## Expected Solution
Visualy, we should be able to create the below infrastructure from terraform, in one command. 
![solution Architecture](./schema.png?raw=true "Two Webservers behind a Load Balancer")

The two webserver-images we are using for this playground are baked with some very simple PHP code, to display their local IP.
# Requirements

# AWS required knowledge

## Step 1 : Define the Provider

## Step 2 : Create one EC2 instance

### Define the Provider

Before Terraform can create resources, it needs to know which platform to connect to.
Terraform support most of the public cloud provider, including AWS.
In our case,  we want terraform to connect to AWS using existing AWS keys.

In the code below, we explicitly define which  Access and Secret Key to use and which AWS region we want to create our resources in.

```
provider "aws" {
  access_key = "<ACCESS_KEY>"
  secret_key = "<SECRET_KEY>"
  region     = "eu-central-1"
}
```
Here we want to  create resources in the region *eu-central-1*, which is the Frankfurt region.

### Create our first ressources, an EC2 instance and its security group
The first resource we want to create, as a part of our infrastructure as code configuration, is an EC2 instance.
For the this playground, Forest technologies has prepared a custom Amazon Machine image *ami-e67c8d89*.

This AMI is available as a community AMI, and is publicly available. It is a Ubuntu Server image, running an Apache WebServer.


```
resource "aws_instance" "web1" {
  ami           = "ami-e67c8d89"
  instance_type = "t2.micro"
  security_groups = ["${aws_security_group.default.name}"]
}
```

Here we decide on which AMI to use, the instance size of the new instance, and its security group.

>security_groups = ["${aws_security_group.default.name}"]

This line is what is called an implicit dependency. Using a dynamic reference, Terraform will automatically fill this value, **only when a security group has been created**.
See next step for the security group.

## Step 3 : Add Security Group

Security group is a AWS level firewall, it comes on top of the machine firewall.
It maps what port are open for inbound (**ingress**) and outbound(**egress**) traffic.

The structure of this ressource is very simple:  we give it a name, simply list _ingress_ or _egress_ blocks that describe the network permissions.

As an example, let's have a look at this SSH ingress block :
```
#SSH access from anywhere
ingress {
  from_port = 22
  to_port = 22
  protocol = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```
**In clear text, this block allows any inbound TCP connection from port 22 to port 22, from  any target IP address.**

Let's add the below to our playground.tf:
```
resource "aws_security_group" "default" {
  name = "SG_example"
  description = "Used in the terraform"

  # SSH access from anywhere
  ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from anywhere
  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # outbound internet access
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
Here, we simply allow SSH and HHTP connections to our instance.


## Step 4 : Apply the template.

In the same directory as our template is, let's run :


`terraform plan`

then

`terraform apply`



## Step 5: Create a second instance
In the same terraform template, let's add a a new resource block to add a second EC2 instance.

```
resource "aws_instance" "web2" {
  ami           = "ami-e67c8d89"
  instance_type = "t2.micro"
  security_groups = ["${aws_security_group.default.name}"]
}
```

On the fly, we should be able to do `terraform apply` again and it should modify our infrastructure transparently.

## Step 6: Create an ELB

The final piece of our infrastructure is a load balancer. AWS provides us with great way to set up load balancer, with ELB.
Luckily, Terraform support ELB perfectly.

To be clear about ELB, it requires  a few things :
1- Availability zones
2- Security group
3- Listener config
4- Health check config
5- List of instances handled by ELB

All of these are implicit dependencies to existing resources in the same template .


```
resource "aws_elb" "web" {
  name = "example-elb"

  # The same availability zone as our instance
  availability_zones = ["${aws_instance.web1.availability_zone}"]
  security_groups = ["${aws_security_group.elb.id}"]
  listener {
    instance_port = 80
    instance_protocol = "http"
    lb_port = 80
    lb_protocol = "http"
  }

  health_check {
    healthy_threshold = 2
    unhealthy_threshold = 2
    timeout = 3
    target = "HTTP:80/"
    interval = 30
  }

  # The instance is registered automatically
  instances = ["${aws_instance.web1.id}","${aws_instance.web2.id}"]

  cross_zone_load_balancing = true
  idle_timeout = 400
  connection_draining = true
  connection_draining_timeout = 400

}
```
#### ... and its security Group
```

resource "aws_security_group" "elb" {
  name = "elb_sg"
  description = "Used in the terraform"

  # HTTP access from anywhere
  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # outbound internet access
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Once this is in the terraform template, `terraform plan` and then `terraform apply`
