# 05 Creating a Kubernetes cluster

Now that we got familiar with our sample application, learned how to
dockerize things and build a CI/CD pipeline, we are going to step things
up a bit. Running our containers locally is great for development and
testing, but eventually we want to make applications available to other
people, no matter if it's for customer-facing or only used internally.
We could go and order a server, install Docker and execute a
`docker run` on that server. It would be possible, but there are
solutions that are far better suited.

## Introducing: Kubernetes

Kubernetes is an open-source platform for managing containerized
services that supports both declarative configurations and automation.
What does that mean? We can manage containerized services,
which is exactly what we've built in the last step. Kubernetes works in a way that we can
write configurations that describe what we want our applications to look
like and Kubernetes takes the necessary steps to get to that point. One
great advantage is that it is completely independent of a particular
infrastructure provider. Basically, you can run Kubernetes on any
infrastructure. The big cloud providers offer managed version of
Kubernetes, which makes the setup much easier. In our case, we are going
to use Amazon EKS (_Elastic Container Service for Kubernetes_), but you
could also use Microsoft's solution called _Azure Kubernetes Service_ or
Google's solution called _Google Kubernetes Engine_.

## Kubernetes Basics

The main idea of Kubernetes is coordinating work on multiple instances,
which are called nodes. The nodes, together with the Kubernetes master
form a cluster. The master is responsible for the management of the
cluster. It coordinates all activities, such as scheduling applications,
scaling applications or rolling out new updates. A node is a virtual
machine or physical computer, in our case simply an EC2 instance. Each
node has a Kubelet, which is an agent for managing the node and
communicating with the Kubernetes master. Additionally, the node has
tools for managing containers, in our case Docker. To make your
applications highly available, you should have more than one node. This
way your application can still be available, even if one node fails. The
graphic below shows the basic concept of the Kubernetes master and three
nodes inside a cluster:

![Kubernetes_master_nodes](../assets/images/module_01_cluster.svg)

Don't worry, you don't need to understand all of the Kubernetes details,
that's far beyond the scope of the DevOps training. As with most things,
the best way to learn about new things is to try them. So let's build
our own cluster now!

## Building a Kubernetes cluster

Clone the repository with the Kubernetes setup scripts:

```bash
cd ~/devops-training
git clone https://github.com/senacor/devops-training-infrastructure
```

Open the file `eks-setup/terraform/eks/variables.tf`. We will specify a prefix here that we can use to distinguish the resources of the training participants. Choose a unique prefix (e.g. your initials or your CWID) in line 7. If you leave the prefix blank, a random prefix will be generated for you.

You will also need to enter the external IP of your workstation in the variable `workstation-external-ip`, to enable access to your Kubernetes cluster from this IP address. Open the following page from your workstation browser and replace the IP address `127.0.0.1` in line 13 with the IP address it shows: [http://ipv4.icanhazip.com/](http://ipv4.icanhazip.com/)

**_devops-training-infrastructure/terraform/eks/variables.tf:_**

```ruby
#
# Variables Configuration
#

variable "resource-prefix" {
  # Insert your unique prefix here (e.g. initials or CWID)
  default = ""
  type    = string
}

variable "workstation-external-ip" {
  # Insert the IP of your workstation here
  default = "127.0.0.1"
  type    = "string"
}
```

When you've specified everything correctly, we are ready to create our
EKS cluster. The terraform configuration is split into multiple files.
The two files that contain the definitions of our resources are:

- eks-cluster.tf - contains the resource definition for the managed
  EKS cluster (the Kubernetes Master)
- eks-worker-nodes.tf - contains the resource definition for the
  cluster nodes

Let's take a look at the interesting parts:

**_devops-training-infrastructure/terraform/eks/eks-cluster.tf:_**

```ruby
...
resource "aws_eks_cluster" "devops" {
  name     = "terraform-eks-${local.resource_prefix}"
  role_arn = "${aws_iam_role.devops-cluster.arn}"

  vpc_config {
    security_group_ids = ["${aws_security_group.devops-cluster.id}"]
    subnet_ids         = data.aws_subnet_ids.subnets.ids
  }

  depends_on = [
    "aws_iam_role_policy_attachment.devops-cluster-AmazonEKSClusterPolicy",
    "aws_iam_role_policy_attachment.devops-cluster-AmazonEKSServicePolicy",
  ]
}
```

The lines above are the "heart" of our cluster setup:

- We define an EKS cluster resource with the name `terraform-eks-<prefix>` (line 75)
- We created a security group that allow communication with the
  cluster API and communication with the worker nodes, which we attach
  to the cluster (line 79)
- We specify the subnets that our cluster will live in (line 80)

That creates our Kubernetes Master. But without nodes, we won't be able
to compute anything. So let's take a look at the resource definition for
the worker nodes:

**_devops-training-infrastructure/terraform/eks/eks-worker-nodes.tf:_**

```ruby
...
resource "aws_launch_configuration" "devops" {
  associate_public_ip_address = true
  iam_instance_profile        = "${aws_iam_instance_profile.devops-node.name}"
  image_id                    = "${data.aws_ami.eks-worker.id}"
  instance_type               = "m4.large"
  name_prefix                 = "terraform-eks-devops"
  security_groups             = ["${aws_security_group.devops-node.id}"]
  user_data_base64            = "${base64encode(local.devops-node-userdata)}"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "devops" {
  desired_capacity     = 2
  launch_configuration = "${aws_launch_configuration.devops.id}"
  max_size             = 2
  min_size             = 1
  name                 = "terraform-eks-${local.resource_prefix}-asg"
  vpc_zone_identifier  = data.aws_subnet_ids.subnets.ids

  tag {
    key                 = "Name"
    value               = "terraform-eks-devops"
    propagate_at_launch = true
  }

  tag {
    key                 = "kubernetes.io/cluster/terraform-eks-${local.resource_prefix}"
    value               = "owned"
    propagate_at_launch = true
  }
}
```

We are launching the nodes in a so-called Autoscaling Group. First, we
define a launch configuration (lines 122 - 134), where we define the
properties of our instance:

- the Amazon Machine Image we want them to be based on
- the instance size
- the security groups
- a bootstrap script that we want to run when the instance gets
  launched

We provide that launch configuration to our autoscaling group. The
autoscaling group will create the number of instances that we desire,
which is 2 (see line 137), but it can automatically scale the number of
instances up or down, if necessary. The instances created by the
Autoscaling Group are the worker nodes that will join our Kubernetes
cluster.

## Let's create the cluster

Now that we have an overview of the resources that make up our cluster,
let's go ahead and create it. Terraform can plan the resources it will
create before actually creating them. We can take a look at the plan by
running:

```bash
cd ~/devops-training/devops-training-infrastructure/terraform/eks
terraform init
terraform plan
```

It will output every resource that will be created, which are 18 in
total, consisting of:

- 5 security group rules
- 2 security groups
- 1 launch configuration
- 1 instance profile
- 1 autoscaling group
- 2 IAM roles
- 5 IAM role policy attachments
- 1 EKS cluster

Let's go ahead and create all these resources:

```bash
terraform apply
```

Type `yes` when asked. Creating a Kubernetes cluster will take a while
(approximately 10 minutes). Go grab a coffee or tea, tell your colleague
how awesome Terraform is and when you come back, you should see the
following output:

```yaml
Apply complete! Resources: 22 added, 0 changed, 0 destroyed.

Outputs:

config_map_aws_auth =

apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::604370441254:role/terraform-eks-tg-node-iam-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

kubeconfig =

apiVersion: v1
clusters:
- cluster:
    server: https://2CD4E92A0E3AC7D937995E19BB1C5565.gr7.us-east-1.eks.amazonaws.com
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRFNU1EVXhOVEV5TXpFek5Gb1hEVEk1TURVeE1qRXlNekV6TkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTWJEClp4b29ya01wTjQ3TDFUL1pqZy95UzFVL29USnFjSlFWeDJsT3c4ZGNhWnliL1ZlSjFPdStuWUduMGhDaHlIdGQKUVkvRldaVnZxRjVCZ2FHbURiNTl5R3VDOTNhVjJyajdPc2w0QjUxWmpROEh6cUJ4ZThiK0xjdllwdmZpbm55bgpuZjFiVWJXTzE4WG9qeStiUDZvdzk3WXFSdkJWSUxtdjBEQ0ppLzIyMGxNMVNNNTJvZU5uczE3WFQrYTM1bXo1CkhsZU9JVVdFaHVDRndERFIzK2NFNzFsVktPejNFSnFac2IrTW1iT2FFSzlBN2JGWEJoaGJobVZBblJDeTllUE8KcVBYSUZhb293bCs4NHZoeE5yV0FlbjdIRThCK21Nc3J4ODV2eDduSjliNXdkNHdYQTNFY3QvenpjRU55ZURWcQpuY3FrNGxlYkc5VG1EQW1rQktzQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFKTDIyMk5ISm0vYllhTjJQZEJEWUhUM0VyUHUKOTNPUm9CeFVUU1ZaWnJ3aCt1dUhKV05FSk1jWnpQQ2ZIRTRWSWc4aGhtYy95Vm5IQzBiRTNGWGxRSUpVSVQ3Mgp5aDhtU0pOa1ZoWFZYb3I0Vml3VVlBc3loVFpFV3ZSSWJNM0NFOHZYTkJjOEJFQks5dExSZWxQbVNsTGZzTnJjCjc4U1FZVDJjODc5dGxmWU50M2w1blB0bGgzeGdVbHBpaDJnbXdYMkdBRzlxeStLR0pGZTRSaUJ4NDdQM0VDMHAKc0FGYXFxS0xsZE5scDdEWFNMT1lQUWtQdTJ0TEM1NG53aGt4dG1RZXhCbzh3YUxqb3h3V2swczBkd2hrUkdRMwoyUkFyWERwcDN0KzB2aUVYTGxWUG51QjhWYk13OTRYMVdwaU9BQSs3aldNczRHZXBML1ZNU3hwUThaaz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: aws
  name: aws
current-context: aws
kind: Config
preferences: {}
users:
- name: aws
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "terraform-eks-tg"

secret-key = <some-secret-key>
```

The output consists of two important things:

- a ConfigMap that allows our worker nodes to join the cluster
- a kubeconfig that allows us to interact with the Kubernetes cluster
  via the kubectl CLI

Let's store the ConfigMap:

```bash
terraform output config_map_aws_auth > config_map_aws_auth.yaml
```

We can store the kubeconfig from the `terraform output` or we can use an AWS CLI command to get our kubeconfig file. Since we are going to use that AWS CLI command later on in our pipeline, we will use it now, too. Update the following command with the name of your Kubernetes cluster, which is the following: `terraform-eks-<prefix>`

```bash
aws eks --region eu-central-1 update-kubeconfig --name <your_kubernetes_cluster>
```

Before we can apply the ConfigMap, we need to edit it. By default, it
adds a role mapping that enables access to the cluster for the user that
created it. Later, we are going to communicate with our cluster from our
CI/CD pipeline. Therefore we need to give the user that we created for
our Gitlab pipeline access, too. Open the file `config_map_aws_auth.yaml` and add the following lines to the end of it. Make sure that you insert the user ARN of your IAM user for Gitlab. You can get it from the AWS IAM console, it looks something like `arn:aws:iam::XXXXXXXXXXXX:user/gitlab-YY`

```yaml
mapUsers: |
  - userarn: <your Gitlab user ARN>
    username: gitlab
    groups:
      - system:masters
```

Overall, the ConfigMap will look like that:

**_devops-training-infrastructure/terraform/eks/config_map_aws_auth.yaml:_**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: <your role ARN>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: <your Gitlab user ARN>
      username: gitlab
      groups:
        - system:masters
```

Now we can apply the ConfigMap to the cluster:

```bash
kubectl apply -f config_map_aws_auth.yaml
```

Run the following command to make sure that the nodes are joining the
cluster:

```bash
kubectl get nodes --watch
```

You should see that two nodes join the cluster. Initially, the status is
`NotReady`, but after a few seconds, the status will change to `Ready`:

```bash
NAME                         STATUS   ROLES    AGE   VERSION
ip-10-0-0-254.ec2.internal   Ready    <none>   52s   v1.12.7
ip-10-0-1-18.ec2.internal    Ready    <none>   54s   v1.12.7
```

Great, you did it! You successfully provisioned a Kubernetes cluster
with two nodes. Give yourself a clap on the shoulder and then keep
going, because the prettiest Kubernetes cluster is worthless if you
don't use it! ;)
