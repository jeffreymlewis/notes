# Overview
This article will explain how to upgrade an AWS EKS cluster.

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
kubectl get nodes
kubectl -n kube-system describe deployment coredns | grep Image
kubectl -n kube-system describe daemonset kube-proxy | grep Image
kubectl -n kube-system describe daemonset aws-node | grep Image
```

## Upgrade Kubernetes Control Plane
Update the eks control plane using the AWS Console.

## Confirm Kubernetes Component Upgrade
Upgrade kubernetes components as described by Amazon https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html

Refer to the “Check Current Kubernetes Versions” section above for viewing current versions. If necessary, manually upgrade k8s components as follows.

### Update kube-proxy
```
kubectl -n kube-system set image daemonset.apps/kube-proxy \
  kube-proxy=602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/kube-proxy:v1.14.9
kubectl -n kube-system describe daemonset kube-proxy | grep Image
# Above command will update the daemonset definition. K8s will then bounce the pods
# one at a time. Confirm all pods are running the correct version as follows.
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o jsonpath="{.items[*].spec.containers[*].image}" | \
  tr -s '[[:space:]]' '\n' ; echo
```

### Update coredns
```
kubectl -n kube-system set image deployment.apps/coredns \
  coredns=602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/coredns:v1.6.6
kubectl -n kube-system describe deployment coredns | grep Image
# K8s will then update pods, one at a time. Confirm all pods running new version.
kubectl -n kube-system get pods -l k8s-app=kube-dns -o jsonpath="{.items[*].spec.containers[*].image}" | \
  tr -s '[[:space:]]' '\n' ; echo
```

### Update aws-node
```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.5/config/v1.5/aws-k8s-cni.yaml
kubectl -n kube-system describe daemonset aws-node | grep Image
# K8s will then update pods, one at a time. Confirm all pods running new version.
kubectl -n kube-system get pods -l k8s-app=aws-node -o jsonpath="{.items[*].spec.containers[*].image}" | \
  tr -s '[[:space:]]' '\n' ; echo
```

## Upgrade EKS Worker Nodes to get new version of kubelet

In this section we will upgrade the EKS Worker nodes.

* Disable Kubernetes Cluster Autoscale, so it doesn’t try to “fix things” when new “unneeded” EKS Workers start appearing.
```
kubectl -n kube-system scale deployments/cluster-autoscaler-aws-cluster-autoscaler --replicas=0
kubectl -n kube-system get pods -l app.kubernetes.io/instance=cluster-autoscaler  # re-run until all pods are shutdown
```

* Replace the EKS Worker nodes. If applications are written properly, and correct Pod Disruption Budgets are set, this should be a "zero-downtime deployment."
```
# check current EKS Worker node versions
kubectl get nodes

# Get list of ASGs
aws autoscaling describe-auto-scaling-groups | jq -r '.AutoScalingGroups[].AutoScalingGroupName'

# Double size of all ASGs
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> | \
  jq -r '.AutoScalingGroups[] | "Desired=\(.DesiredCapacity) Max=\(.MaxSize) Min=\(.MinSize)"'

aws autoscaling set-desired-capacity \
  --auto-scaling-group-name <asg-name> \
  --desired-capacity <num>

# cordon/drain nodes
kubectl cordon <node1> <node2> ... <nodeN>
kubectl drain --ignore-daemonsets --delete-emptydir-data <node1>
kubectl drain --ignore-daemonsets --delete-emptydir-data <node2>
...
kubectl drain --ignore-daemonsets --delete-emptydir-data <nodeN>

# Ensure all pods are rescheduled
kubectl get po -A | grep Pending

# remove old nodes
autoscaling terminate-instance-in-auto-scaling-group --should-decrement-desired-capacity --instance-id <instance-id>
```

* Check one node (or more, if you’re paranoid) to confirm the correct version of kubelet.
```
kubectl get nodes
ssh $upgraded_node
kubelet --version
```

* Restart cluster autoscaler
```
kubectl -n kube-system scale deployments/cluster-autoscaler-aws-cluster-autoscaler --replicas=0
kubectl -n kube-system get pods -l app.kubernetes.io/instance=cluster-autoscaler  # re-run until all pods are shutdown
```
