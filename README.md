# devopsplayground6-terraform


# Overview
This Devops Playground will explore Infrastructure as Code with Hashicorp Terraform and AWS. 
The goal is to automate the creation of AWS resources, using terraform. In order to give an overview of Terraform's features, we will try to launch two WebServer instances, and put them behind a load balancer, transparently, through a single, unified configuration file.

## Expected Solution
Visualy, we should be able to create the below infrastructure from terraform, in one command. 
![solution Architecture](./schema.png?raw=true "Two Webservers behind a Load Balancer")

The two webserver-images we are using for this playground are baked with some very simple PHP code, that displays their local IP.
# Requirements

* You should have beeen provided with a VM : a simple Ubuntu 16.04, on which Terraform has been installed.
* AWS access keys: here again, they should be baked into the provided VM. See Step 1 to reproduce it with your own keys, if you wish.
* Access to an AWS Machine Image, custom made for this playground. This will be a simple webserver image.


# AWS required knowledge

* Elastic Cloud Compute (EC2) : A server.
* Security Group : AWS-level Firewall
* Amazon Machine Image (AMI) : Image template used to create instance from. We aim to use _ami-e67c8d89_ for the Frankfurt zone; alternatively in the Ireland zone _ami-57e19724_ is also available.
* Elastic Load Balancer (ELB) : AWS-specific load balancer

## Step 0 : On the Virtual Machine

On the VM, there should be a `playground/` folder, in which we will find *playground.tf*.

This playground will entirely be done in this *playground.tf* file.
Using your favorite text editor, edit this file.

## Step 1 : Define the Provider

Before Terraform can create resources, it needs to know which platform to connect to.
Terraform supports most of the public cloud providers, including AWS.
In our case,we want terraform to connect to AWS using existing AWS keys.

In the code below, we explicitly define which Access and Secret Key to use and which AWS region we want to create our resources in.

```
provider "aws" {
  access_key = "<ACCESS_KEY>"
  secret_key = "<SECRET_KEY>"
  region     = "eu-central-1"
}
```
Here we want to  create resources in the region *eu-central-1*, which is the Frankfurt region.

## Step 2 : Create the Security Group for the Webservers

Security group is an AWS level firewall, it comes on top of the machine firewall.
It maps which ports are open for inbound (**ingress**) and outbound(**egress**) traffic.

The structure of this ressource is very simple: we give it a name and simply list _ingress_ or _egress_ blocks describing the network permissions.

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
**This block allows any inbound TCP connection from port 22 to port 22, from  any target IP address.**

Let's add the below to our _playground.tf_:
**Make sure to replace _<yourname>_ with your name.**
```
resource "aws_security_group" "SG-Webserver-<yourname>" {
  name = "SG-Webserver-<yourname>"
  description = "Security Group used for the Webservers."

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

## Step 3 : Create one EC2 instance

The second resource we want to create, as a part of our infrastructure as code configuration, is an EC2 instance.
For the this playground, Forest technologies has prepared a custom Amazon Machine image *ami-e67c8d89*.

This AMI is available as a community AMI, and is publicly available. It is an Ubuntu Server image, running an Apache WebServer.

Let's copy this code into our _playground.tf_.
*Make sure to replace _<yourname>_ with your name.*
```
resource "aws_instance" "Webserver1-<yourname>" {
  ami           = "ami-e67c8d89"
  instance_type = "t2.micro"
  security_groups = ["${aws_security_group.SG-Webserver-<yourname>.name}"]
  tags {
    Name = "Webserver1-<yourname>"
  }
}

output "dns-webserver-1" {
    value = "${aws_instance.Webserver1-<yourname>.public_dns}"
}
```

Here we decide on which AMI to use, the instance size of the new instance, and its security group.

>security_groups = ["${aws_security_group.SG-Webserver-<yourname>.name"]

This line is what is called an implicit dependency. Using a dynamic reference, Terraform will automatically find the _name_ value of the _SG-Webserver-<yourname>_ resource.

The output block will output the public DNS of the instance once Terraform will have created the instance.

## Step 4 : Apply the configuration.

In the directory containing our *playground.tf* configuration file, let's run :

Terraform can simulate the effects of a configuration change, before applying them. It is checking the current state of the infrastructure (which is inexistant at this stage) and lists the operations that need to be performed to follow the new configuration; some ressources will be added, edited or deleted.

`terraform plan`

If we are satisfied with the plan, we can _apply_ the changes to AWS, and terraform will create all the resources in the correct order.

`terraform apply`

This operation can take several minutes to finish.

If everything is successful, the output of this first run should be _dns-webserver-1_, let's try to connect to this address.

## Step 5: Create a second instance
In the same terraform configuration file, let's create a new resource block to add a second EC2 instance to our infrastructure.

*Make sure to replace _<yourname>_ with your name.*
```
resource "aws_instance" "Webserver2-<yourname>" {
  ami           = "ami-e67c8d89"
  instance_type = "t2.micro"
  security_groups = ["${aws_security_group.SG-Webserver-<yourname>.name}"]
  tags {
    Name = "Webserver2-<yourname>"
  }
}

output "dns-webserver-2" {
    value = "${aws_instance.Webserver2-<yourname>.public_dns}"
}
```

On the fly, we should be able to do `terraform apply` again and it should update our infrastructure transparently.

## Step 6: Create an ELB and its security group
### Security Group first
Here again, we need to create a security group  for the ELB to work properly; this one follows the exact same logic as the one above.
Let's copy it into our _playground.tf_.

*Make sure to replace _<yourname>_ with your name.*

```
resource "aws_security_group" "SG-ELB-<yourname>" {
  name = "SG-ELB-<yourname>"
  description = "Security Group used for the ELB."

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

### And finally the load balancer

The final piece of our infrastructure is a load balancer. AWS provides us with great way to set up load balancer, with ELB.
Luckily, Terraform support ELB perfectly.

To be clear about ELB, it requires  a few things :
* Availability zones
* Security group
* Listener config : Which port is the Load Balancer serving ?
* Health check config : Which port and URI should the ELB check on ?
* List of instances handled by ELB

Let's copy and paste the code below in our _playground.tf_
*Make sure to replace _<yourname>_ with your name.*

```
resource "aws_elb" "ELB-<yourname>" {
  name = "ELB-<yourname>"

  # The same availability zone as our instance
  availability_zones = ["${aws_instance.Webserver1-<yourname>.availability_zone}"]
  security_groups = ["${aws_security_group.SG-ELB-<yourname>.id}"]
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
  instances = ["${aws_instance.Webserver1-<yourname>.id}","${aws_instance.Webserver2-<yourname>.id}"]

  cross_zone_load_balancing = true
  idle_timeout = 400
  connection_draining = true
  connection_draining_timeout = 400

}

output "dns-ELB" {
    value = "${aws_elb.ELB-<yourname>.dns_name}"
}
```

Once this is in the terraform configuration, `terraform plan` and then `terraform apply`

## Step 7: Destroying the infrastructure
Using `terraform destroy` allows you to destroy everything managed by the terraform configuration in a single command.

#Next steps
This configuration can of course be improved in many ways:
* Store variables in a different file, making the configuration more flexible.
* Store the output configuration in a different file, allowing easier modification.
* Use a secret management tool, such as vault, to safely store sensitive information (AWS keys)


#Reading materials
* [Terraform doc on AWS](https://www.terraform.io/docs/providers/aws/)
* [AWS blog post on Terraform](https://aws.amazon.com/blogs/apn/terraform-beyond-the-basics-with-aws/)
* [Terraform best practices](https://github.com/hashicorp/best-practices)
* [Forest Technologies' Blog](http://www.forest-technologies.co.uk/blog)
