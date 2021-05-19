# Overview
This article will explain how to upgrade a Kubernetes cluster in Amazon EKS. This procedure can also be used to swap out the nodes (EC2 instances), to change instance size, etc.

# Instructions

## Read the documentation

AWS EKS versions: https://docs.aws.amazon.com/eks/latest/userguide/platform-versions.html

Amazon Upgrade Instructions: https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html

Manual Upgrade Instructions for CoreDNS (if needed): https://docs.aws.amazon.com/eks/latest/userguide/coredns.html

## Install new 'kubectl'

You'll need a new version of `kubectl`. 

## Check Current Kubernetes Versions

Record the current Kubernetes versions for later use.

```
kubectl version --short
kubectl -n kube-system describe deployment coredns | grep Image
kubectl -n kube-system describe daemonset kube-proxy | grep Image
kubectl -n kube-system describe daemonset aws-node | grep Image
kubectl get nodes
```

## Upgrade Kubernetes Control Plane

## Confirm Kubernetes Component Upgrade

Upgrade kubernetes components as described by Amazon https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html

Refer to the “Check Current Kubernetes Versions” section above for viewing current versions. If necessary, manually upgrade k8s components at this time.

### Manual Kubernetes Component Upgrade

If manual upgrade of the kubernetes components is necessary, follow these steps.

**kube-proxy**
```
kubectl -n kube-system set image daemonset.apps/kube-proxy \
  kube-proxy=602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/kube-proxy:v1.14.9
kubectl -n kube-system describe daemonset kube-proxy | grep Image
# Above command will update the daemonset definition. K8s will then bounce the pods
# one at a time. Confirm all pods are running the correct version as follows.
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o jsonpath="{.items[*].spec.containers[*].image}" | \
  tr -s '[[:space:]]' '\n' ; echo
```

**coredns**
```
kubectl -n kube-system set image deployment.apps/coredns \
  coredns=602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/coredns:v1.6.6
kubectl -n kube-system describe deployment coredns | grep Image
# K8s will then update pods, one at a time. Confirm all pods running new version.
kubectl -n kube-system get pods -l k8s-app=kube-dns -o jsonpath="{.items[*].spec.containers[*].image}" | \
  tr -s '[[:space:]]' '\n' ; echo
```

**aws-node**
```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.5/config/v1.5/aws-k8s-cni.yaml
kubectl -n kube-system describe daemonset aws-node | grep Image
# K8s will then update pods, one at a time. Confirm all pods running new version.
kubectl -n kube-system get pods -l k8s-app=aws-node -o jsonpath="{.items[*].spec.containers[*].image}" | \
  tr -s '[[:space:]]' '\n' ; echo
```

## Upgrade EKS Worker Nodes to get new version of kubelet

In this section we will upgrade the EKS Worker nodes.

1. Disable Kubernetes Cluster Autoscale, so it doesn’t try to “fix things” when new “unneeded” EKS Workers start appearing.
1. Disable Cluster Autoscale
```
kubectl -n kube-system scale deployments/cluster-autoscaler-aws-cluster-autoscaler --replicas=0
kubectl -n kube-system get pods -l app.kubernetes.io/instance=cluster-autoscaler  # re-run until all pods are shutdown
```
1. Upgrade EKS Worker nodes. If applications are written properly, and correct Pod Disruption Budgets are set, this should be a “zero-downtime deployment.”
1. Replace EKS Worker nodes

# check current EKS Worker node versions
kubectl get nodes

# Rolling restart of all nodes
aws autoscaling describe-auto-scaling-groups | jq -r .AutoScalingGroups[].AutoScalingGroupName | grep -E ^app-workers-eks | while read asg_name; do
  echo "###"
  echo "### $asg_name"
  echo "###"
  kubergrunt eks deploy --region us-west-2 --asg-name $asg_name
  sleep 15
done

Check one node (or more, if you’re paranoid) to confirm the correct version of kubelet.

Double-check kubelet version

kubectl get nodes
ssh $upgraded_node
kubelet --version

Upgrade & Enable Cluster Autoscale

https://github.com/kubernetes/autoscaler/releases

cd infrastructure-live/$env/$region/$env/services/eks-core-services
git checkout master ; git pull ; git checkout -b eks-upgrade-$env
vim terragrunt.hcl
terragrunt apply

# terraform should update desired version
kubectl -n kube-system describe deployment cluster-autoscaler-aws-cluster-autoscaler | grep Image

# Deploy two replicas
kubectl -n kube-system scale deployments/cluster-autoscaler-aws-cluster-autoscaler --replicas=2
kubectl -n kube-system get pods -l app.kubernetes.io/instance=cluster-autoscaler

# Check CA versions
kubectl -n kube-system get pod $(kubectl -n kube-system get pods -l app.kubernetes.io/instance=cluster-autoscaler -o jsonpath='{.items[0].metadata.name}') -o yaml | grep image

# Check logs on each node
kubectl -n kube-system logs $(kubectl -n kube-system get pods -l app.kubernetes.io/instance=cluster-autoscaler -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $(kubectl -n kube-system get pods -l app.kubernetes.io/instance=cluster-autoscaler -o jsonpath='{.items[1].metadata.name}')

Related articles


