# EKS Cluster on Spot Instances

## Cluster Configuration

This project contains EKS cluster configuration file, that can help creating a new EKS cluster running on top of spot instances (regular and GPU-powered)

The EKS cluster contains 4 node groups:

1. `spot-ng` node group - auto scaling, multi-AZ, scale from 2 to 20, for running `kube-system` and app workloads
2. `gpu-spot-ng-a|b|c` 3 node groups (different AZ) - single-AZ, scale from 0, GPU-powered, taints to evict non-GPU workload

## Create EKS cluster

Create a new EKS cluster with above configuration:

```sh
eksctl create -f cluster.yaml
```

## NVIDIA device plugin for Kubernetes

The [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin) for Kubernetes is a Daemonset that allows you to automatically:

- Expose the number of GPUs on each nodes of your cluster
- Keep track of the health of your GPUs
- Run GPU enabled containers in your Kubernetes cluster

Install NVIDIA device plugin `Daemonset` into the `kube-system` namespace and only on GPU-powered nodes (using `nodeAffinity` with `beta.kubernetes.io/instance-type` key).

```sh
kubectl install -f kubernetes/nvidia-device-plugin.yaml
```

## Cluster Autoscaler

The [cluster autoscaler](https://github.com/kubernetes/autoscaler) on AWS scales worker nodes within any specified autoscaling group. It will run as a `Deployment` in your cluster.

To run a cluster-autoscaler which auto-discovers ASGs with nodes use the `--node-group-auto-discovery` flag.
In the current setup the `--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled` flag is used.

```sh
kubectl create -f kubernetes/cluster-autoscaler.yaml
```

## (optional) Manage EKS nodes with SSM Agent

In order to manage EKS nodes and access terminal without exposing node over SSH, it's recommended to use AWS Systems Manager.

Usually SSM agent is installed during node bootstrap. But I suggest to use SSM Agent deployed as `Daemonset`, which does not require to install anything on EKS node and can be easily uninstalled (just delete the SSM agent `Daemonset`).

Follow guide from [kube-ssm-agent](https://github.com/alexei-led/kube-ssm-agent) repository.

```sh
kubectl create -f daemonset.yaml
```

## 