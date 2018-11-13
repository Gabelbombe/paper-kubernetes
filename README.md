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
