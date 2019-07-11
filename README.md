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

### Session to EKS node

First, list all EKS nodes with instance types and instance IDs.

```sh
kubectl k get nodes --output="custom-columns=NAME:.metadata.name,ID:.spec.providerID,TYPE:.metadata.labels.beta\.kubernetes\.io\/instance-type"

NAME                                            ID                                      TYPE
ip-192-168-102-82.us-west-2.compute.internal    aws:///us-west-2a/i-00ca7c066c89291e6   p2.xlarge
ip-192-168-103-6.us-west-2.compute.internal     aws:///us-west-2a/i-0d4737e4e56baa5c5   p3.2xlarge
ip-192-168-108-201.us-west-2.compute.internal   aws:///us-west-2a/i-0b70fe641d4d5e93a   c4.4xlarge
ip-192-168-134-221.us-west-2.compute.internal   aws:///us-west-2b/i-064b071e6047b9524   p2.8xlarge
ip-192-168-189-140.us-west-2.compute.internal   aws:///us-west-2c/i-04753b8b515e9ea35   c4.4xlarge
```

Then, select any instance and open a shell to it:

```sh
aws ssm start-session --target i-0d4737e4e56baa5c5

Starting session with SessionId: ....
sh-4.2$
```

## Run GPU workload

The generated EKS cluster has 3 GPU-powered node groups that can be scaled from 0 to serve requested GPU tasks and get back to 0 when work is completed.

### How does it work

Cluster autoscaler uses `k8s.io/cluster-autoscaler/enabled` ASG tag for auto-discovery across multiple AZ zones (using `--balance-similar-node-groups` flag).

Cluster autoscaler scales workload based on requested resource, `nvidia/gpu` for GPU workload.

All GPU EKS node are protected with `taint`: `nvidia.com/gpu: "true:NoSchedule"` and in order to run a GPU workload on these nodes, the workload must define a corresponding `tollerations`, like following:

```yaml
...
tolerations:
  - key: "nvidia.com/gpu"
    operator: "Exists"
    effect: "NoSchedule"
...
```
