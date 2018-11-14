# Let's build Kubernetes from A-Z with Terraform and Ansible

## _Part 1: The Infrastructure_

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

The code snippets have been simplified. Please refer to the [code repository](https://github.com/ehime/terraform-kubernetes/tree/master/terraform) for the complete version.


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

I'm using an official AMI for Ubuntu 16.04 to keep it simple. Here is, for example, the definition of `etcd` instances.

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

Instances have a static private IP address. I'm using a simple address pattern to make them human-friendly: 10.43.0.1x are etcd instances, `10.43.0.2x` Controllers and `10.43.0.3x` Workers (aka Kubernetes Nodes or Minions), but this is not a requirement.

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

The interesting point here is the template-based generation of certificates. I'm using [Data Sources](https://www.terraform.io/docs/configuration/data-sources.html), a feature introduced by Terraform 0.7.


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

All components use the same certificate, so it has to include all addresses (IP and/or DNS names). In a real-world project, we would use stable DNS names, and the certificate would include them only.

CFSSL command line utility generates `.pem` files for CA certificate, server key and certificate. In the following articles, we will see how they are uploaded into all machines and used by Kubernetes CLI to connect to the API.


### Known simplifications and limitations

Let's sum up the most significant simplifications introduced in this part of the project:

- All instances are in a single, public subnet.
- All instances are directly accessible from outside the VPC. No ÂVPN, no Bastion (though, traffic is allowed only from a single, configurable IP).
- Instances have static internal IP addresses. Any real-world environment should use DNS names and, possibly, dynamic IPs.
- A single server certificate for all components and nodes.



## _Part 2: The Provisioner_

### Installing Kubernetes

We have at this point, created all AWS resources using Terraform. No Kubernetes component has been installed yet.

We have 9 EC instances (hosts, in Ansible terms), 3 of each type (group, for Ansible):

- **Controllers**: Kubernetes HA master
  - Will run Kubernetes API Server, Controller Manager and Scheduler services
- **Workers**: Kubernetes Nodes or Minions
  - Will run Docker, Kubernetes Proxy and Kubelet services
  - Will have [CNI](https://github.com/containernetworking/cni) installed for networking between containers
- **etcd**: etcd 3 nodes cluster to maintain Kubernetes state

All hosts need the certificates we generated, for HTTPS.

<img src='assets/k8snthw-components.png' />
<br />
<br />


First of all, we have to install Python 2.5+ on all machines.


### Ansible project organisation

The Ansible part of the project is organised as [suggested by Ansible documentation](http://docs.ansible.com/ansible/playbooks_best_practices.html#directory-layout). We also have multiple playbooks, to run independently:

- Bootstrap Ansible (install Python). Install, configure and start all the required components (`infra.yml`)
- Configure Kubernetes CLI (kubectl) on your machine (`kubectl.yml`)
- Setup internal routing between containers (`kubernetes-routing.yml`)
- Smoke test it, deploying a nginx service (`kubernetes-nginx.yml`) + manual operations

This section walks through the first playbook (`infra.yml`).

The code snippets have been simplified. Please refer to the [code repository](https://github.com/ehime/terraform-kubernetes/tree/master/ansible) for the complete version.


### Installing Kubernetes components

The first playbook takes care of bootstrapping Ansible and installing Kubernetes components. Actual tasks are separate in roles: `common` (executed on all hosts); one role per machine type: `controller`, `etcd` and `worker`.

Before proceeding, we have to understand how Ansible identifies and find hosts.


### Dynamic Inventory

Ansible works on groups of hosts. Each host must have a unique handle and address to SSH into the box.

The most basic approach is using a [Static Inventory](http://docs.ansible.com/ansible/intro_inventory.html#hosts-and-groups), a hardwired file associating groups to hosts and specifying the IP address (or DNS name) of each host.

A more realistic approach uses a [Dynamic Inventory](http://docs.ansible.com/ansible/intro_dynamic_inventory.html), and a static file to define groups, based on instance tags. For AWS, Ansible provides an [EC2 AWS Dynamic Inventory script](http://docs.ansible.com/ansible/intro_dynamic_inventory.html#example-aws-ec2-external-inventory-script) out-of-the-box.

The configuration file `ec2.ini`, downloaded from Ansible repo, requires some change. It is very long, so here are the modified parameters only:

```ini
[ec2]
instance_filters         = tag:ansibleFilter=Kubernetes01
regions                  = us-east-1
destination_variable     = ip_address
vpc_destination_variable = ip_address
hostname_variable        = tag_ansibleNodeName
```

Note we use instance tags to filter and identify hosts and we use IP addresses for connecting to the machines.

A separate file defines groups, based on instance tags. It creates nicely named groups, `controller`, `etcd` and `worker` (we might have used groups weirdly called `tag_ansibleNodeType_worker`...). If we add new hosts to a group, this file remains untouched.

```ini
[tag_ansibleNodeType_etcd]
[tag_ansibleNodeType_worker]
[tag_ansibleNodeType_controller]

[etcd:children]
tag_ansibleNodeType_etcd

[worker:children]
tag_ansibleNodeType_worker

[controller:children]
tag_ansibleNodeType_controller
```

To make the inventory work, we put Dynamic Inventory Python script and configuration file in the same directory with groups file.

```bash
ansible/
  hosts/
    ec2.py
    ec2.ini
    groups
```

The final step is configuring Ansible to use this directory as inventory. In `ansible.cfg`:

```ini
[defaults]
...
inventory = ./hosts/
...
```


### Bootstrapping Ansible

Now we are ready to execute the playbook (`infra.yaml`) to install all components. The first step is installing Python on all boxes with a raw module. It executes a shell command remotely, via SSH, with no bell and whistle.

```yml
- hosts: all
  gather_facts: false

  tasks:
  - name: Update distros
    raw: "apt-get -y update"
    become: true
    retries: 10
    delay: 20

  - name: Install Python
    raw: "apt-get -y -q install python"
    become: true
```


### Installing and configuring Kubernetes components: Roles

The second part of the playbook install and configure all Kubernetes components. It plays different roles on hosts, depending on groups. Note that groups and roles have identical names, but this is not a general rule.


```yml
- hosts: etcd
  roles:
    - common
    - etcd

- hosts: controller
  roles:
    - common
    - controller

- hosts: worker
  roles:
    - common
    - worker
```

Ansible executes the common role (omitted here) on all machines. All other roles do the real job. They install, set up and start services using `systemd`:

- Copy the certificates and key (the ones we generated with Terraform)
- Download service binaries directly from the official source, unpack and copy them to the right directory
- Create the _systemd_ unit file, using a template
- Bump both _systemd_ and the service
- Verify the service is running

Here are the tasks of `etcd` role . Other roles are not substantially different.

```yml
- name: Create etcd config dir
  file: path=/etc/etcd state=directory
  become: true

- name: Copy certificates
  copy:
    src: "{{ playbook_dir }}/../cert/{{ item }}"
    dest: "/etc/etcd/"
  become: true
  with_items:
    - ca.pem
    - kubernetes.pem
    - kubernetes-key.pem

- name: Download etcd binaries
  get_url:
    url: "https://github.com/coreos/etcd/releases/download/v3.0.1/etcd-v3.0.-linux-amd64.tar.gz"
    dest: "/usr/local/src"
  become: true

- name: Unpack etcd binaries
  unarchive:
    copy: no
    src: "/usr/local/src/etcd-v3.0.-linux-amd64.tar.gz"
    dest: "/usr/local/src/"
    creates: "/usr/local/src/etcd-v3.0.-linux-amd64/etcd"
  become: true

- name: Copy etcd binaries
  copy:
    remote_src: true
    src: "/usr/local/src/etcd-v3.0.-linux-amd64/{{ item }}"
    dest: "/usr/bin"
    owner: root
    group: root
    mode: 0755
  with_items:
    - etcd
    - etcdctl
  become: true

- name: Create etcd data dir
  file: path=/var/lib/etcd state=directory
  become: true

- name: Add etcd systemd unit
  template:
    src: etcd.service.j2
    dest: /etc/systemd/system/etcd.service
    mode: 700
  become: true

- name: Reload systemd
  command: systemctl daemon-reload
  become: true

- name: Enable etcd service
  command: systemctl enable etcd
  become: true
Â
- name: Restart etcd
  service:
    name: etcd
    state: restarted
    enabled: yes
  become: true

- name: Wait for etcd listening
  wait_for: port=2379 timeout=60

- name: Verify etcd cluster health
  shell: etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
  register: cmd_result
  until: cmd_result.stdout.find("cluster is healthy") != -1
  retries: 5
  delay: 5
```

We generate `etcd.service` from a template. Ports are hardwired (may be externalised as variables), but hosts IP addresses are _[facts](http://docs.ansible.com/ansible/playbooks_variables.html#information-discovered-from-systems-facts)_ gathered by Ansible.

```ini
# {{ ansible_managed }}
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/bin/etcd --name {{ inventory_hostname }} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --initial-advertise-peer-urls https://{{ ansible_eth0.ipv4.address }}:2380 \
  --listen-peer-urls https://{{ ansible_eth0.ipv4.address }}:2380 \
  --listen-client-urls https://{{ ansible_eth0.ipv4.address }}:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://{{ ansible_eth0.ipv4.address }}:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster {% for node in groups['etcd'] %}{{ node }}=https://{{ hostvars[node].ansible_eth0.ipv4.address }}:2380{% if not loop.last %},{% endif %}{% endfor %} \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```


### Known simplifications

The most significant simplifications, compared to a real world project, concern two aspects:

- Ansible workflow is simplistic: at every execution, it restarts all services. In a production environment, you should add guard conditions, trigger operations only when required (e.g. when the configuration has changed) and avoid restarting all nodes of a cluster at the same time.
- As in the first part, using fixed internal DNS names, rather than IPs, would be more realistic.

**Congradulations!** The `infra.yml` playbook has installed and run all the services required by Kubernetes.


## Part 3: Controlling Kubernetes

At this point we have completed installation of Kubernetes components. There is still one important step: setting up the routing between Workers (aka Nodes or Minions) to allow Pods living on different machines to talk each other. As a final smoke test, we’ll deploy a nginx service.

Before starting though, we'll have to configure Kubernetes CLI on our machine to remotely interact with the cluster.


### Inputs from Terraform

For running the following steps, we need to know Kubernetes API ELB public DNS name and Workers public IP addresses. Terraform outputs them at the end of provisioning. In this simplified project, we have to note them down, manually.


### Setup Kubernetes CLI

This step is not part of the platform set up. We configure Kubernetes CLI locally to interact with the remote cluster.

Setting up the client requires running few shell commands. The save the API endpoint URL and authentication details in the local [kubeconfig](http://kubernetes.io/docs/user-guide/kubeconfig-file/) file. They are all local shell commands, but we will use a playbook (`kubectl.yaml`) to run them.

```yml
# Expects `kubernetes_api_endpoint` as `--extra-vars "kubernetes_api_endpoint=xxxx"`
- hosts: 127.0.0.1
  connection: local

  tasks:
  - name: Set kubectl endpoint
    shell: "kubectl config set-cluster {{ cluster_name }} --certificate-authority={{ playbook_dir }}/../cert/ca.pem --embed-certs=true --server=https://{{ kubernetes_api_endpoint }}:6443"

  - name: Set kubectl credentials
    shell: "kubectl config set-credentials {{ user }} --token {{ token }}"

  - name: Set kubectl default context
    shell: "kubectl config set-context default-context --cluster={{ cluster_name }} --user={{ user }}"

  - name: Switch kubectl to default context
    shell: "kubectl config use-context default-context"
```

The client uses the CA certificate, generated by Terraform. User and token must match those in the token file  (`token.csv`), also used for Kubernetes API Server setup. The API load balancer DNS name must be passed to the playbook as a parameter.

```bash
$ ansible-playbook kubectl.yaml --extra-vars "kubernetes_api_endpoint=kube-375210502.us-east-1.elb.amazonaws.com"
```

Kubernetes CLI is now configured, and we may use `kubectl` to control the cluster.

```bash
$ kubectl get componentstatuses

NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
```


### Setup internal routing

Kubernetes uses subnets for networking between Pods. These subnets have nothing to do with the subnet we defined in AWS.

Our VPC subnet is `10.43.0.0/16`, while the Pod subnets are part of `10.200.0.0/16` (`10.200.1.0/24`, `10.200.2.0/24` etc.). We have to setup routes between workers instances for these subnets.

<img src='assets/k8snthw-networking' />

As we are using the `Kubenet` [network plugin](http://kubernetes.io/docs/admin/network-plugins/#kubenet), Pod subnets are dynamically assigned. Kube Controller decides Pod subnets within a Pod Cluster CIDR (defined by 1 parameter on `kube-controller-manager` startup). Subnets are dynamically assigned and we cannot configure these routes at provisioning time, using Terraform. We have to wait until all Kubernetes components are up and running, discover Pod subnets querying Kubernetes API and then add the routes.

In Ansible, we might use the `ec2_vpc_route_table` module to modify AWS Route Tables, but this would interfere with route tables managed by Terraform. Due to its stateful nature, tampering with Terraform managed resources is not a good idea.

The solution (hack?) adopted here is adding new routes directly to the machines, after discovering Pod subnets, using `kubectl`. It is the job of `kubernetes-routing.yml` playbook, the Ansible translation of the following steps:

Query Kubernetes API for Workers Pod subnets. Actual Pod subnets (the second column) may be different, but they are not necessarily assigned following Workers numbering.


```bash
$ kubectl get nodes --output=jsonpath='{range .items[*]}{.status.addresses[?(@.type=="InternalIP")].address}{.spec.podCIDR}{"\n"}{end}'
10.43.0.30 10.200.2.0/24
10.43.0.31 10.200.0.0/24
10.43.0.32 10.200.1.0/24
```

Then, on each Worker, add routes for Pod subnets to the owning Node

```bash
$ sudo route add -net 10.200.2.0 netmask 255.255.255.0 gw 10.43.0.30 metric 1
$ sudo route add -net 10.200.0.0 netmask 255.255.255.0 gw 10.43.0.31 metric 1
$ sudo route add -net 10.200.1.0 netmask 255.255.255.0 gw 10.43.0.32 metric 1
```

... and add an IP Tables rule to avoid internal traffic being routed through the Internet Gateway:

```bash
$ sudo iptables -t nat -A POSTROUTING ! -d 10.0.0.0/8 -o eth0 -j MASQUERADE
```


### Smoke test the system, deploying nginx

The last step is a smoke test. We launch multiple nginx containers in the cluster, then create a Service exposed as NodePort (a random port, the same on every Worker node). The are three local shell commands. The `kubernetes-nginx.yml`) is the Ansible version of them.

```bash
$ kubectl run nginx --image=nginx --port=80 --replicas=3
...
$ kubectl expose deployment nginx --type NodePort
...
$ kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}'
32700
```

The final step is manual (no playbook!). To test the service we fetch the default page from nginx.

All Workers nodes directly expose the Service. Get the exposed port from the last command you run, get Workers public IP addresses from Terraform output.

This should work for all the Workers:

```bash
$ curl http://<worker0-ip-address>:<port>
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

### Known Simplifications

- The way we set routing for Pod networks is hacky and fragile. If a Workers restarts or if we add new Workers, you have to recalculate Pod subnets routes and update them on all Workers. In real production projects, you’d better use a network overlay with [Flannel](https://coreos.com/blog/introducing-rudder/).
- Compared to the [original tutorial](https://github.com/kelseyhightower/kubernetes-the-hard-way), we skipped deploying DNS Cluster add-on.


### Conclusions

There is a lot of space for improvement to make it more realistic, using DNS names, a VPN or a bastion, moving Instances in private subnets. A network overlay (Flannel) would be another improvement. Modifying the project to add these enhancements might be a good learning exercise.
