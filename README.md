# Lotus Cloud Templates

This repository can be used as a bulding block for deploying a Kubernetes cluster and the Lotus Daemon and/or Miner. We provide templates for [Amazon EKS](https://aws.amazon.com/eks/?whats-new-cards.sort-by=item.additionalFields.postDateTime&whats-new-cards.sort-order=desc&eks-blogs.sort-by=item.additionalFields.createdDate&eks-blogs.sort-order=desc) and [Azure AKS](https://azure.microsoft.com/en-us/services/kubernetes-service/), as well as [FluxCD](https://fluxcd.io) integration for the [lotus-aio Helm Chart](https://github.com/Digital-MOB-Filecoin/lotus-charts) as well as the [Kube Prometheus Stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) preconfigured with Grafana dashboards for Lotus.

Mining on testnets with smaller sectors numbers *should* work on top of Kubernetes in most cases. If you wish to mine on Mainnet, we recommend running miners on dedicated hardware that match the [official hardware requirements](https://docs.filecoin.io/mine/hardware-requirements/). 

## Running on AWS

The easiest way to create an EKS cluster is with [eksctl](https://eksctl.io). Created by Weaveworks, eksctl is the official tool to automate the creation of EKS clusters. We provide [a template](eks/eksctl.yaml) that you can adapt according to your own needs. The default values provide just enough resources for running the Lotus Daemon software.

### Hardware recommendations

We recommend using [Storage Optimized instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/storage-optimized-instances.html) that provide NVMe disks, specifically the `i3` or `i3en` class. In our tests, the Lotus Daemon was stable while running on `i3en.2xlarge` instances. Running a miner will require at least `i3en.6xlarge`. 

### eksctl Configuration

The eksctl manifest provided will create an EKS v1.18 cluster in the `us-east-1` region, as well as 3 nodegroups for running your workloads, in `us-east-1a`, `us-east-1b`, respectively `us-east-1c`. It will create a dedicated VPC, enable IAM roles for ServiceAccounts, and configure the GitOps integration for FluxCD, which in turn will automatically install `kube-prometheus-stack` and the `lotus-aio` Helm Chart.

#### Changing the region

If you wish to change the AWS Region, make sure you update `metadata.region` to the desired region, as well as `availabilityZones` and `nodeGroups[*].availaibilityZones` to match your new region.

#### Changing the instance type

By default we are using `i3en.2xlarge` instance types. If you wish to use a different instance type, update `nodeGroups[*].instanceType`.

#### Public cluster nodes

EKS is configured to run the worker nodes in a private subnet. In this case, if you wish to have external connectivity to the Lotus nodes you will have to expose them via a `LoadBalancer` service, which the `lotus-aio` Helm Chart does by default. However, you can also configure your EKS worker nodes to run on the Public VPC, thus having public IP addresses assigned to them. To do this, set `nodeGroups[*].privateNetworking=false`.

#### NVMe provisioning

There are 2 options for using NVMe drives as local storage:

1. (Recommended) Use `preBootstrapCommands` to format your NVMe drives. This is not included by default because each instance type has a different number of NVMe drives. Please make sure you check the specification of your chosen instance type. Failure to format and mount your drives correctly will render the worker nodes unable to join the EKS cluster. With this method, you can mount the local path inside your container, and you will have to use `taints` to make sure that the lotus-aio Pod is always scheduled to the same node.

2. [eks-nvme-ssd-provisioner](https://github.com/brunsgaard/eks-nvme-ssd-provisioner) is a NVMe provisioner created by the community that is able to format NVMe drives. We already provide the required labels in the eksctl manifest, if needed.

