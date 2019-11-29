# Teardown

Running all these resources that we provisioned costs money, so you need to make sure to clean them up after you're done with the training.

We first need to delete our workload from the Kubernetes cluster. If we don't do it before removing the Kubernetes cluster, the LoadBalancers that we provisioned will not get deleted.

```bash
kubectl delete --all namespaces
```

Next, we can delete the ECR repository that we created with Terraform:

```bash
cd ~/devops-training/terraform-demo/
terraform destroy
```

Enter `yes` when asked.

Then you can delete the EKS cluster with the following command:

```bash
cd ~/devops-training/devops-training-infrastructure/terraform/eks
terraform destroy
```

Enter `yes` when asked.
