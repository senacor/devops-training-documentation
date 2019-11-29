# Prerequisites

In order to follow the instructions of the training, you will need a few things.

1. Access to an AWS account
2. An IDE to edit code
3. The following tools installed on your local maching:
   - `git`
   - `docker` and `docker-compose`
   - `terraform`
   - `node` and `npm``
4. A Gitlab account (you can use a free account, you just need to create one)

Throughout the instructions you will see a few `cd` commands, that will help you navigate through the provided resources. We assume that you clone all necessary git repositories into `~/devops-training`. If you work in another folder on your machine, everything will still work, but you will need to adjust paths in all `cd` commands.

## VPC setup

For your Terraform cluster, you will need to create a VPC. This is not part of the training, as you might be provided with a VPC beforehand. If this is not the case, use the following instructions to create a VPC:

Clone the necessary git repository:

```bash
git clone https://github.com/senacor/devops-training-infrastructure
cd devops-training-infrastructure/terraform/vpc
terraform init
terraform apply
```

## AWS IAM group setup

Create an IAM group called `CICD` and attach the following two policies to it:

- `AmazonEC2ContainerServiceFullAccess`
- `AmazonEC2ContainerRegistryFullAccess`

Additionally, you need to attach a policy that allows EKS access. You can create a policy that looks like this and attach it:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["eks:*"],
      "Resource": "*"
    }
  ]
}
```

All gitlab IAM users are added to the group, so that the Gitlab CI/CD pipeline can push to the ECR registry.
