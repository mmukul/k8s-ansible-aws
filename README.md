# Provisioning a Kubernetes Cluster on AWS using Ansible

## Setup

- AWS VPC
- 3 EC2 instances for HA Kubernetes Control Plane: Kubernetes API, Scheduler and Controller Manager
- 3 EC2 instances for *etcd* cluster
- 3 EC2 instances as Kubernetes Workers (aka Minions or Nodes)
- Kubenet Pod networking (using CNI)
- HTTPS between components and control API
- Sample *nginx* service deployed to check everything works


## Requirements

Requirements on control machine:

- Python (tested with Python 2.7.12, may be not compatible with older versions; requires Jinja2 2.8)
- Python *netaddr* module
- Ansible (tested with Ansible 2.1.0.0)
- *cfssl* and *cfssljson*:  https://github.com/cloudflare/cfssl
- Kubernetes CLI
- SSH Agent


## AWS Credentials

### AWS KeyPair

You need a valid AWS Identity (`.pem`) file and the corresponding Public Key. Terraform imports the [KeyPair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) in your AWS account. Ansible uses the Identity to SSH into machines.


### Ansible authentication

Ansible expect AWS credentials set in environment variables:

```
$ export AWS_ACCESS_KEY_ID=<access-key-id>
$ export AWS_SECRET_ACCESS_KEY="<secret-key>"
```


Ansible expects the SSH identity loaded by SSH agent:

```
$ ssh-add <keypair-name>.pem
```

## Defining the environment

- `control_cidr`: The CIDR of your IP. All instances will accept only traffic from this address only. Note this is a CIDR, not a single IP. e.g. `123.45.67.89/32` (mandatory)
- `default_keypair_public_key`: Valid public key corresponding to the Identity you will use to SSH into VMs. e.g. `"ssh-rsa AAA....xyz"` (mandatory)

**Note that Instances and Kubernetes API will be accessible only from the "control IP"**. If you fail to set it correctly, you will not be able to SSH into machines or run Ansible playbooks.

You may optionally redefine:

- `default_keypair_name`: AWS key-pair name for all instances.  (Default: "k8s-not-the-hardest-way")
- `vpc_name`: VPC Name. Must be unique in the AWS Account (Default: "kubernetes")
- `elb_name`: ELB Name for Kubernetes API. Can only contain characters valid for DNS names. Must be unique in the AWS Account (Default: "kubernetes")
- `owner`: `Owner` tag added to all AWS resources. No functional use. It becomes useful to filter your resources on AWS console if you are sharing the same AWS account with others. (Default: "kubernetes")


## Install Kubernetes with Ansible

Run Ansible commands from `./ansible` subdirectory.


## Install Kubernetes components and *etcd* cluster.

```
$ ansible-playbook infra.yaml
```

### Setup Kubernetes CLI

Configure Kubernetes CLI (`kubectl`) on your machine, setting Kubernetes API endpoint.

```
$ ansible-playbook kubectl.yaml --extra-vars "kubernetes_api_endpoint=<kubernetes-api-dns-name>"
```

Verify all components and minions (workers) are up and running, using Kubernetes CLI (`kubectl`).

```
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}

$ kubectl get nodes
NAME                                       STATUS    AGE
ip-10-43-0-30.eu-west-1.compute.internal   Ready     6m
ip-10-43-0-31.eu-west-1.compute.internal   Ready     6m
ip-10-43-0-32.eu-west-1.compute.internal   Ready     6m
```

### Setup Pod cluster routing

Set up additional routes for traffic between Pods.

```
$ ansible-playbook kubernetes-routing.yaml
```

## Deploy a *ngnix* service inside Kubernetes.

```
$ ansible-playbook kubernetes-nginx.yaml
```

Verify pods and service are up and running.

```
$ kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-2032906785-9chju   1/1       Running   0          3m        10.200.1.2   ip-10-43-0-31.eu-west-1.compute.internal
nginx-2032906785-anu2z   1/1       Running   0          3m        10.200.2.3   ip-10-43-0-30.eu-west-1.compute.internal
nginx-2032906785-ynuhi   1/1       Running   0          3m        10.200.0.3   ip-10-43-0-32.eu-west-1.compute.internal

> kubectl get svc nginx --output=json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "nginx",
        "namespace": "default",
...
```

Retrieve the port *nginx* has been exposed on:

```
$ kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}'
32700
```

Now you should be able to access *nginx* default page:

```
$ curl http://<worker-0-public-ip>:<exposed-port>
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```
