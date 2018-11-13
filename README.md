# K8s from scratch to AWS with Terraform and Ansible

### The goal

The purpose of this series of articles is presenting a simple, but realistic example of how to provision a Kubernetes cluster on AWS, using Terraform and Ansible. This is an educational tool, not a production-ready solution nor a simple way to quickly deploy Kubernetes.

I will walk through the sample project, describing the steps to automatise the process of provisioning simple Kubernetes setup on AWS, trying to make clear most of the simplifications and corners cut. To follow it, you need some basic understanding of AWS, Terraform, Ansible and Kubernetes.

The complete working project is available here: [terraform-kubernetes](https://github.com/ehime/terraform-kubernetes). Please, read the documentation included in the repository, describing requirements and step by step process to execute it.


### Starting point

This Kubernetes setup inspired by _[Kubernetes the Hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way)_, by [Kelsey Hightower](https://github.com/kelseyhightower) from Google. The original tutorial is for Google Cloud. My goal is demonstrating how to automate the process on AWS, so I "translated" it to AWS and transformed the manual steps into Terraform and Ansible code.


### Target platform

<div style='width:800px; margin: 0 auto;'>
  <div style='width: 200px; height: 400px; float: left;'>
  Our target Kubernetes cluster includes:

  - 3 EC2 instances for Kubernetes Control Plane: _Controller Manager, Scheduler, API Server._
  - 3 EC2 instances for the HA etcd cluster.
  - 3 EC2 instances as Kubernetes workers (aka Minions or Nodes, in Kubernetes jargon)
  - Container networking using Kubenet plugin (relying on [CNI](https://github.com/containernetworking/cni))
  - HTTPS communication between all components
  - All Kubernetes and etcd components run as services directly in the VM (not in Containers).
  </div>
  <img style='float:right;' src='assets/k8snthw-infrastructure.png' />
</div>

<div style="clear:both; padding-top:200px"/>


### Automation Tools

For our example I'll be using [Terraform](https://www.terraform.io/intro/index.html) and [Ansible](http://docs.ansible.com/ansible/intro.html) for many reasons:

- Terraform declarative approach works very well in describing and provisioning infrastructure resources, while it is very limited when you have to install and configure.
- Ansible procedural approach is very handy for installing and configuring software on heterogeneous machines, but become hacky when you use it for provisioning the infrastructure;
- Terraform + Ansible is the stack I often use for production projects in in DevOps.

Terraform and Ansible overlap. I will use Terraform to provision infrastructure resources, then pass the baton to Ansible, to install and configure software components.


### Terraform to provision infrastructure

Terraform allow us to describe the target infrastructure; then it takes care to create, modify or destroy any required resource to match our blueprint. Regardless its declarative nature, Terraform allows some [programming patterns](https://github.com/ehime/paper-designpatterns/blob/master/README.md). In this project, resources are in grouped in files; constants are externalised as variables and we will use of templating. We are not going to use Terraform Modules.

The code snippets have been simplified. Please refer to the [code repository](https://github.com/ehime/terraform-kubernetes) for the complete version.


### Create VPC and networking layer

After specifying the AWS provider and the region (omitted here), the first step is defining the VPC, the single subnet and an Internet Gateway.

```hcl
resource "aws_vpc" "kubernetes" {
  cidr_block           = "10.43.0.0/16"
  enable_dns_hostnames = true
}

resource "aws_subnet" "kubernetes" {
  vpc_id            = "${aws_vpc.kubernetes.id}"
  cidr_block        = "10.43.0.0/16"
  availability_zone = "us-east-1a"
}
```

To make the subnet public, let's add an Internet Gateway and route all outbound traffic through it:

```hcl
resource "aws_internet_gateway" "gw" {
  vpc_id = "${aws_vpc.kubernetes.id}"
}

resource "aws_route_table" "kubernetes" {
    vpc_id = "${aws_vpc.kubernetes.id}"
    route {
      cidr_block = "0.0.0.0/0"
      gateway_id = "${aws_internet_gateway.gw.id}"
    }
}

resource "aws_route_table_association" "kubernetes" {
  subnet_id = "${aws_subnet.kubernetes.id}"
  route_table_id = "${aws_route_table.kubernetes.id}"
}
```

We also have to import the Key-pair that will be used for all Instances, to be able to SSH into them. The Public Key must correspond to the Identity file loaded into SSH Agent (please, see the [README](https://github.com/ehime/terraform-kubernetes/blob/master/README.md) for more details)
