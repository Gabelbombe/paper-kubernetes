# Let's build Kubernetes from A-Z with Terraform and Ansible

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

```hcl
resource "aws_key_pair" "default_keypair" {
  key_name   = "my-keypair"
  public_key = "ssh-rsa AA....zzz"
}
```


### Create EC2 Instances

I’m using an official AMI for Ubuntu 16.04 to keep it simple. Here is, for example, the definition of `etcd` instances.

```hcl
variable "deletion_protection" {
  description = "The database can't be deleted when this value is set to true."
  default     = false
}

resource "aws_instance" "etcd" {
  count         = 3
  ami           = "ami-1967056a" // Unbuntu 16.04 LTS HVM, EBS-SSD
  instance_type = "t2.micro"

  subnet_id                   = "${aws_subnet.kubernetes.id}"
  private_ip                  = "${cidrhost("10.43.0.0/16", 10 + count.index)}"
  associate_public_ip_address = true

  availability_zone      = "us-east-1a"
  vpc_security_group_ids = ["${aws_security_group.kubernetes.id}"]
  key_name               = "my-keypair"
}
```

Other instances, `controller` and `worker`, are no different, except for one important detail: Workers have `source_dest_check = false` to allow sending packets from IPs not matching the IP assigned to the machine by AWS (for Inter-Container communication).

```hcl
resource "aws_iam_role" "kubernetes" {
  name = "kubernetes"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [ { "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" } ]
}
EOF
}

resource "aws_iam_role_policy" "kubernetes" {
  name = "kubernetes"
  role = "${aws_iam_role.kubernetes.id}"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    { "Action" : ["ec2:*"], "Effect": "Allow", "Resource": ["*"] },
    { "Action" : ["elasticloadbalancing:*"], "Effect": "Allow", "Resource": ["*"] },
    { "Action": "route53:*", "Effect": "Allow",  "Resource": ["*"] },
    { "Action": "ecr:*", "Effect": "Allow", "Resource": "*" }
  ]
}
EOF
}

resource "aws_iam_instance_profile" "kubernetes" {
  name  = "kubernetes"
  roles = ["${aws_iam_role.kubernetes.name}"]
}
```

### Static vs. dynamic IP address vs. internal DNS

Instances have a static private IP address. I’m using a simple address pattern to make them human-friendly: 10.43.0.1x are etcd instances, `10.43.0.2x` Controllers and `10.43.0.3x` Workers (aka Kubernetes Nodes or Minions), but this is not a requirement.

A static address is required to have a fixed "handle" for each instance. In a big project, you have to create and maintain a “map” of assigned IPs  and be careful to avoid clashes. It sounds easy, but it could become messy in a big project. On the flip side, dynamic IP addresses change if (when) VMs restart for any uncontrollable event (hardware failure, the provider moving to different physical hardware, etc.), therefore DNS entry must be managed by the VM, not by Terraform... but this a different story.

Real-world projects use internal DNS names as stable handles, not static IP. But to keep this project simple, I will use static IP addresses, assigned by Terraform, and no DNS.


### Installing Python 2.x?

Ansible requires Python 2.5+ on managed machines. Ubuntu 16.04 comes with Python 3 that not compatible with Ansible and  we have to install it before running any playbook.

Terraform has a `remote-exec` provisioner. We might execute `apt-get install python...` on the newly provisioned instances. But the provisioner is not very smart. So, the option I adopted is making Ansible _"pulling itself over the fence by its bootstraps"_, and install Python with a raw module.


### Resource tagging

Every resource has multiple tags assigned (omitted in the snippets, above):

- `ansibleFilter`: fixed for all instances (`"Kubernetes01"` by default), used by Ansible Dynamic Inventory to filter instances belonging to this project.
- `ansibleNodeType`: define the type (group) of the instance, e.g. `"worker"`. Used by Ansible to group different hosts of the same type.
- `ansibleNodeName`: readable, unique identifier of an instance (e.g. `"worker2"`). Used by Ansible Dynamic Inventory as replacement of hostname.
- `Name`: identifies the single resource (e.g. `"worker-2"`). No functional use, but useful on AWS console.
- `Owner`: your name or anything identifying you, if you are sharing the same AWS account with other teammates. No functional use, but handy to filter your resources on AWS console.

```hcl
resource "aws_instance" "worker" {
    count = 3
    ...
    tags {
      Owner           = "Hackathon"
      Name            = "worker-${count.index}"
      ansibleFilter   = "Kubernetes01"
      ansibleNodeType = "worker"
      ansibleNodeName = "worker${count.index}"
    }
  }
}
```

### A load balancer for Kubernetes API

For High Availability, we have multiple instances running Kubernetes API server and we expose the control API using an external Elastic Load Balancer.

```hcl
resource "aws_elb" "kubernetes_api" {
  name                      = "kube-api"
  instances                 = ["${aws_instance.controller.*.id}"]
  subnets                   = ["${aws_subnet.kubernetes.id}"]
  cross_zone_load_balancing = false

  security_groups = ["${aws_security_group.kubernetes_api.id}"]

  listener {
    lb_port           = 6443
    instance_port     = 6443
    lb_protocol       = "TCP"
    instance_protocol = "TCP"
  }

  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 15
    target              = "HTTP:8080/healthz"
    interval            = 30
  }
}
```

The ELB works at TCP level (layer 4), forwarding connection to the destination. The HTTPS connection is terminated by the service, not by the ELB. The ELB need no certificate.

### Security

The security is very simplified in this project. We have two Security Groups: one for all instances and another for the Kubernetes API Load Balancer (some rule omitted here).

```hcl
resource "aws_security_group" "kubernetes" {
  vpc_id = "${aws_vpc.kubernetes.id}"
  name   = "kubernetes"

  # Allow all outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow all internal
  ingress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    self      = true
  }

  # Allow all traffic from the API ELB
  ingress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = ["${aws_security_group.kubernetes_api.id}"]
  }

  # Allow all traffic from control host IP
  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["${var.control_cidr}"]
  }
}

resource "aws_security_group" "kubernetes_api" {
  vpc_id = "${aws_vpc.kubernetes.id}"
  name   = "kubernetes-api"

  # Allow inbound traffic to the port used by Kubernetes API HTTPS
  ingress {
    from_port   = 6443
    to_port     = 6443
    protocol    = "TCP"
    cidr_blocks = ["${var.control_cidr}"]
  }

  # Allow all outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

All instances are directly accessible from outside the VPC: not acceptable for any production environment. But security is not entirely lax: inbound traffic is allowed only from one IP address: the public IP address you are connecting from. This address is configured by the Terraform variable `control_cidr`. Please, read the [project documentation](https://github.com/ehime/terraform-kubernetes/blob/master/README.md) for further explanations.

No matter how lax, this configuration is tighter than the default security set up by [Kubernetes cluster creation script](http://kubernetes.io/docs/getting-started-guides/aws/).


### Self-signed Certificates

Communication between Kubernetes components and control API, all use HTTPS. We need a server certificate for it. It is self-signed with our own private CA. Terraform generates a CA certificate, a server key+certificate and signs the latter with the CA. The process uses [CFSSL](https://github.com/cloudflare/cfssl).

The interesting point here is the template-based generation of certificates. I’m using [Data Sources](https://www.terraform.io/docs/configuration/data-sources.html), a feature introduced by Terraform 0.7.


```hcl
# Generate Certificates
data "template_file" "certificates" {
  template   = "${file("${path.module}/template/kubernetes-csr.json")}"
  depends_on = ["aws_elb.kubernetes_api", "aws_instance.etcd", "aws_instance.controller", "aws_instance.worker"]

  vars {
    kubernetes_api_elb_dns_name = "${aws_elb.kubernetes_api.dns_name}"
    kubernetes_cluster_dns      = "${var.kubernetes_cluster_dns}"
    etcd0_ip                    = "${aws_instance.etcd.0.private_ip}"
    ...
    controller0_ip              = "${aws_instance.controller.0.private_ip}"
    ...
    worker2_ip                  = "${aws_instance.worker.2.private_ip}"
  }
}

resource "null_resource" "certificates" {
  triggers {
    template_rendered = "${ data.template_file.certificates.rendered }"
  }

  provisioner "local-exec" {
    command = "echo '${ data.template_file.certificates.rendered }' > ../cert/kubernetes-csr.json"
  }

  provisioner "local-exec" {
    command = "cd ../cert; cfssl gencert -initca ca-csr.json | cfssljson -bare ca; cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes"
  }
}
```
