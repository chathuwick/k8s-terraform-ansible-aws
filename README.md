# k8s-terraform-ansible-aws
Creating kubernetes cluster in AWS using terraform and ansible
<!-- TOC -->

- [Platform](#platform)
    - [Kubernetes Cluster Architecture](#kubernetes-cluster-architecture)
    - [Tools](#tools)
    - [Requirements](#requirements)
- [AWS Credentials](#aws-credentials)
    - [AWS keypair](#aws-keypair)
    - [AWS Access Key and Secret Access Key](#aws-access-key-and-secret-access-key)
- [Setting up the Environment](#setting-up-the-environment)
        - [You may optionally redefine:](#you-may-optionally-redefine)
    - [Changing AWS Region](#changing-aws-region)
- [Provisioning the Infrastructure](#provisioning-the-infrastructure)
    - [Generated SSH config](#generated-ssh-config)
- [Configure Kubernetes using Ansible](#configure-kubernetes-using-ansible)
    - [Setup Kubernetes CLI](#setup-kubernetes-cli)
    - [Setup Pod cluster routing](#setup-pod-cluster-routing)
- [Deploy Nodejs App](#deploy-nodejs-app)
- [Deployment Pipeline](#deployment-pipeline)

<!-- /TOC -->
## Platform
### Kubernetes Cluster Architecture
- EC2 instance for kubernetes controller including kube API server, scheduler and controller manager
- EC2 instance for ETCD cluster
- EC2 worker instances including kubelet and kube-proxy
Container networking using kubernets plugin

![alt tag](https://github.com/chathuwick/k8s-terraform-ansible-aws/blob/master/img/architecture.png)
### Tools
- **Terraform:** Provisioning Infrastructure
- **Ansible:** Installing and configuring software

### Requirements
- Ubuntu server (Run terraform and ansible scripts)
- [Terraform](https://www.terraform.io/downloads.html) (tested with 0.7.0)
- Python (tested with Python 2.7.12; require Jinja2-2.10)
- Python netaddr module (tested with netaddr-0.7.19)
Install boto SDK
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) (tested with Ansible 2.1.0.0)
- [cfssl](http://www.pimwiddershoven.nl/entry/install-cfssl-and-cfssljson-cloudflare-kpi-toolkit) and [cfssljson](http://www.pimwiddershoven.nl/entry/install-cfssl-and-cfssljson-cloudflare-kpi-toolkit)
- kubectl
````
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.6.6/bin/linux/amd64/kubectl
````
- Install docker (to build docker images using ansible)
- Login to docker registry (to push the docker image to docker registry)
- SSH Agent

## AWS Credentials
### AWS keypair
Create the AWS keypair and the public key in the AWS region that you desired to create the kuberenetes cluster

### AWS Access Key and Secret Access Key
Both Terraform and Ansible expect AWS credentials set in environment variables:
````
$ export AWS_ACCESS_KEY_ID=<access-key-id>
$ export AWS_SECRET_ACCESS_KEY=<secret-access-key>
````
Ansible expects the SSH identity loaded by SSH agent:
````
$ ssh-add <keypair-name>.pem
````
## Setting up the Environment
Terraform expects some variables to define your working environment:

- **control-cidr:** The CIDR of your IP. All instances will accept only traffic from this address only. Note this is a CIDR, not a single IP. e.g. _123.45.67.89/32_ (mandatory)
- **default_keypair_public_key:** Valid public key corresponding to the Identity you will use to SSH into VMs. e.g. "ssh-rsa AAA....xyz" (mandatory)
````
$ ssh-keygen –y –f <keypair-name>.pem
````
**Note that Instances and Kubernetes API will be accessible only from the "control IP".** If you fail to set it correctly, you will not be able to SSH into machines or run Ansible playbooks

#### You may optionally redefine:

- **default_keypair_name:** AWS key-pair name for all instances. (Default: "k8s-not-the-hardest-way")
- **vpc_name:** VPC Name. Must be unique in the AWS Account (Default: "kubernetes")
- **elb_name:** ELB Name for Kubernetes API. Can only contain characters valid for DNS names. Must be unique in the AWS Account (Default: "kubernetes")
- **owner:** Owner tag added to all AWS resources. No functional use. It becomes useful to filter your resources on AWS console if you are sharing the same AWS account with others. (Default: "kubernetes")

The easiest way is creating a terraform.tfvars variable file in ./terraform directory. Terraform automatically imports it.
Sample terraform.tfvars:

````
default_keypair_public_key = "ssh-rsa AAA...zzz"
control_cidr = "123.45.67.89/32"
default_keypair_name = "chathuKey"
vpc_name = "Chathu VPC"
elb_name = "Chathu ELB"
owner = "Chathu"
````
### Changing AWS Region
By default, the project uses eu-west-1. To use a different AWS Region, set additional Terraform variables:

- **region:** AWS Region (default: "eu-west-1").
- **zone:** AWS Availability Zone (default: "eu-west-1a")
- **default_ami:** Pick the AMI for the new Region from https://cloud-images.ubuntu.com/locator/ec2/: Ubuntu 16.04 LTS (xenial), HVM:EBS-SSD

You also have to edit ./ansible/hosts/ec2.ini, changing regions = eu-west-1 to the new Region.

## Provisioning the Infrastructure

Run Terraform commands from ./terraform subdirectory.
```
$ terraform plan
$ terraform apply
```
Terraform outputs public DNS name of Kubernetes API and Workers public IPs.

![alt tag](https://github.com/chathuwick/k8s-terraform-ansible-aws/blob/master/img/terraformOutput.png)

### Generated SSH config

Terraform generates ssh.cfg, SSH configuration file in the project directory. It is convenient for manually SSH into machines using node names (controller0, etcd0..2, worker0..2), but it is NOT used by Ansible.
````
$ ssh -F ssh.cfg worker0
````
## Configure Kubernetes using Ansible
- Run Ansible commands from ./ansible subdirectory.
- Install and set up Kubernetes cluster
- Install Kubernetes components and etcd cluster.
````
ansible-playbook infra.yaml
````

### Setup Kubernetes CLI
Configure Kubernetes CLI (kubectl) on your machine, setting Kubernetes API endpoint (as returned by Terraform).
````
$ ansible-playbook kubectl.yaml --extra-vars "kubernetes_api_endpoint=<kubernetes-api-dns-name>"
````
### Setup Pod cluster routing
Set up additional routes for traffic between Pods.
````
$ ansible-playbook kubernetes-routing.yaml
````
## Deploy Nodejs App
Deploy nodejs app in kubernets
````
$ ansible-playbook nodejs-deployment.yaml
````

- This ansible playbook requires some values to pass which can be found in roles/deployment/vars/main.yml
- git_repo: git repository of your nodejs app
- git_branch: git branch name
- docker_img_tag_name: tag name of your docker image which is uploading to your docker registry
- k8s_deployment_name: deployment object name to creat the deployment in kubernetes cluster
- k8s_deployment_port: nodejs app running port
- k8s_deployment_replicas: Number of pods you are going to create
- k8s_service_port: App can be access from externaly through this port

Sample roles/deployment/vars/main.yml
**git_repo:** "https://github.com/chathuwick/node-todo.git" \
**git_branch:** "master" \
**docker_img_name:** "nodejsapp" \
**docker_img_tag_name:** "uwickch/k8s-nodejs:V3" \
**k8s_deployment_name:** "nodejsapp" \
**k8s_deployment_port:** "8080" \
**k8s_deployment_replicas:** "1" \
**k8s_service_port:** "80"

## Deployment Pipeline
- Clone your nodejs app from git repo
- Build the docker image
- Push the docker image to the docker hub
- Create deployment object in kubernets cluster
- Expose the deployment  to a service and access the app using external load balancer

Finally you can get the external URL to the deployed nodejs app by executing below command
```
Kubectl describe svc < k8s_deployment_name >
```
![alt tag](https://github.com/chathuwick/k8s-terraform-ansible-aws/blob/master/img/kubectlsvcOutput.png)
