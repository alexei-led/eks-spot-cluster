# EKS with mixed worker nodes

This project contains Cloudformation template to create EKS worker nodes group for existing EKS cluster.
This worker nodes group can be created from mixture of OnDemand and Spot instances.

Once node group is created with Cloudformation template, edit and create a ConfigMap resource with instance role ARN to join newly created nodes to the EKS cluster.

