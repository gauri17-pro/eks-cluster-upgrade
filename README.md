# EKS Cluster Upgrade Guide

A step-by-step guide to safely upgrading an Amazon EKS cluster, including the control plane, managed node groups, self-managed nodes, and add-ons.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Upgrade Strategy Overview](#upgrade-strategy-overview)
3. [Phase 1 — Pre-Upgrade Checks](#phase-1--pre-upgrade-checks)
4. [Phase 2 — Upgrade the Control Plane](#phase-2--upgrade-the-control-plane)
5. [Phase 3 — Upgrade EKS Add-ons](#phase-3--upgrade-eks-add-ons)
6. [Phase 4 — Upgrade Managed Node Groups](#phase-4--upgrade-managed-node-groups)
7. [Phase 5 — Upgrade Self-Managed Node Groups](#phase-5--upgrade-self-managed-node-groups)
8. [Phase 6 — Post-Upgrade Validation](#phase-6--post-upgrade-validation)
9. [Rollback Plan](#rollback-plan)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before starting the upgrade, ensure you have the following in place:

- **AWS CLI** v2.x installed and configured
  https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- **kubectl** matching the target Kubernetes version (±1 minor version)
- **eksctl** (latest version) — optional but recommended

### Tool Version Check

```bash
aws --version
kubectl version --client
eksctl version
helm version
```

## Create an EKS Cluster

- Install EKSCTL

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

```bash
eksctl create cluster --name my-cluster --version 1.34 --region ap-south-1 --nodegroup-name my-ng-1 --node-type t2.medium --nodes 2 
```

---

## Upgrade Strategy Overview

> ⚠️ **EKS supports upgrading only one minor version at a time.**  
> For example: `1.33 → 1.34 → 1.35`. You cannot skip versions.

**Upgrade order (always follow this sequence):**

```
Control Plane  →  Add-ons  →  Node Groups  →  Validation
```

Skipping this order can result in incompatible API versions or broken cluster networking.

---

## Phase 1 — Pre-Upgrade Checks

### 1.1 Review the Kubernetes Changelog

Check the [Kubernetes changelog](https://kubernetes.io/releases/) and [AWS EKS release notes](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html) for deprecated APIs or breaking changes in the target version.

### 1.2 Check Current Cluster Version

```bash
aws eks describe-cluster --name <cluster-name> --query "cluster.version" --output text
```

### 1.3 Identify Deprecated API Usage

Use `kubectl` to detect usage of deprecated APIs in your workloads:

```bash
kubectl get all --all-namespaces -o yaml | grep "apiVersion
```

### 1.4 Check Add-on Compatibility

```bash
aws eks describe-addon-versions --kubernetes-version <target-version>
```

```
aws eks describe-addon-versions --kubernetes-version 1.35 --addon-name coredns \
  --query 'addons[].addonVersions[?addonVersion==`v1.12.4-eksbuild.10`].{Version:addonVersion,Compatibilities:compatibilities}'
```

Verify that all your installed add-ons (CoreDNS, kube-proxy, VPC CNI, etc.) have versions compatible with the target Kubernetes version.


### 1.5 Backup Critical Resources

```bash
# Export all namespace resources
kubectl get all --all-namespaces -o yaml > cluster-backup-$(date +%F).yaml

# Backup ConfigMaps and Secrets
kubectl get configmap,secret --all-namespaces -o yaml >> cluster-backup-$(date +%F).yaml
```

### 1.6 Notify Stakeholders

Communicate a maintenance window to all relevant teams. Cluster upgrades may cause brief API server unavailability (typically < 5 minutes).

---

## Phase 2 — Upgrade the Control Plane

The control plane (API server, scheduler, controller manager) is managed by AWS and upgraded in-place with zero data plane disruption.

### 2.1 Trigger the Control Plane Upgrade

**Via AWS CLI:**

```bash
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.35
```

**Via eksctl:**

```bash
eksctl upgrade cluster --name my-cluster --region ap-south-1 --version 1.35 --approve
```

**Via AWS Console:**

1. Go to **EKS → Clusters → \<cluster-name\>**
2. Click **Update now** next to the Kubernetes version
3. Select the target version and confirm

### 2.2 Monitor the Upgrade

```bash
aws eks describe-cluster --name my-cluster --query "cluster.status"
# Expected output: "UPDATING" → "ACTIVE"
```

Or watch via eksctl:

```bash
eksctl utils describe-stacks --region ap-south-1 --cluster my-cluster
```

> ⏱️ Control plane upgrades typically take **10–25 minutes**.

### 2.3 Confirm Control Plane Version

```bash
kubectl version --short
# Server version should reflect the new version
```

---

## Phase 3 — Upgrade EKS Add-ons

After the control plane is upgraded, update the managed add-ons. Always upgrade add-ons **before** upgrading node groups.

### 3.1 List Installed Add-ons

```bash
aws eks list-addons --cluster-name my-cluster
```

### 3.2 Check Available Add-on Versions

```bash
aws eks describe-addon-versions \
  --addon-name <addon-name> \
  --kubernetes-version <target-version> \
  --query "addons[].addonVersions[].addonVersion"
```

### 3.3 Upgrade Each Add-on

Upgrade the following add-ons in this order:

#### VPC CNI (aws-node)

```bash
eksctl update addon \
  --cluster my-cluster \
  --name vpc-cni \
  --version v1.21.1-eksbuild.7 \
  --force
```

#### CoreDNS

```bash
eksctl update addon \
  --cluster my-cluster \
  --name vpc-cni \
  --version v1.14.2-eksbuild.4 \
  --force 
```

#### kube-proxy

```bash
eksctl update addon \
  --cluster my-cluster \
  --name vpc-cni \
  --version v1.35.3-eksbuild.5 \
  --force 
```

#### Metrics server

```bash
eksctl update addon \
  --cluster my-cluster \
  --name vpc-cni \
  --version v0.8.1-eksbuild.6 \
  --force
```

### 3.4 Monitor Add-on Status

```bash
aws eks describe-addon \
  --cluster-name <cluster-name> \
  --addon-name <addon-name> \
  --query "addon.status"
# Expected: "UPDATING" → "ACTIVE"
```

---

## Phase 4 — Upgrade Managed Node Groups

Managed node groups support rolling updates with automatic cordon and drain behavior.

### 4.1 Check Current Node Group Version

```bash
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --query "nodegroup.releaseVersion"
```

### 4.2 Get Latest AMI Release Version

```bash
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.35/amazon-linux-2/recommended/release_version \
  --query "Parameter.Value" \
  --output text
```

### 4.3 Trigger Node Group Upgrade

**Via AWS CLI:**

```bash
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-ng-1 \
  --release-version <ami-release-version>
```

**Via eksctl:**

```bash
eksctl upgrade nodegroup \
  --name my-ng-1 \
  --cluster my-cluster \
  --kubernetes-version 1.35
```

### 4.4 Monitor Node Group Upgrade

```bash
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --query "nodegroup.status"
# Expected: "UPDATING" → "ACTIVE"
```

Watch node rollout:

```bash
kubectl get nodes -w
```

> ⏱️ Node group upgrades take **5–15 minutes per node**, depending on cluster size.

---

## Phase 5 — Upgrade Self-Managed Node Groups

For self-managed nodes (launched via Launch Templates or CloudFormation), you must manually cordon, drain, and replace nodes.

### 5.1 Cordon All Nodes in the Group

Prevent new pods from being scheduled on old nodes:

```bash
kubectl cordon <node-name>

# Or cordon all nodes in a group at once using a label
kubectl get nodes -l eks.amazonaws.com/nodegroup=<nodegroup-name> -o name | \
  xargs kubectl cordon
```

### 5.2 Drain Each Node

Safely evict all pods from the node:

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --timeout=300s
```

> 💡 `--ignore-daemonsets` is required since DaemonSet pods cannot be evicted.  
> `--delete-emptydir-data` is needed for pods using `emptyDir` volumes.

### 5.3 Update the Launch Template

1. Go to **EC2 → Launch Templates**
2. Create a new version of the template with the updated EKS-optimized AMI for the target version
3. Find the latest AMI:

```bash
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/<target-version>/amazon-linux-2/recommended/image_id \
  --query "Parameter.Value" \
  --output text
```

### 5.4 Terminate Old Nodes

After draining, terminate old nodes so the Auto Scaling Group launches new ones with the updated AMI:

```bash
aws ec2 terminate-instances --instance-ids <instance-id>
```

Or update the Auto Scaling Group to perform an instance refresh:

```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name <asg-name> \
  --preferences '{"MinHealthyPercentage": 90}'
```

### 5.5 Verify New Nodes

```bash
kubectl get nodes
# All nodes should show the new Kubernetes version
```

---

## Phase 6 — Post-Upgrade Validation

### 6.1 Verify Cluster Version

```bash
kubectl version --short
aws eks describe-cluster --name <cluster-name> --query "cluster.version"
```

### 6.2 Check Node Readiness

```bash
kubectl get nodes
# All nodes should show STATUS: Ready
```

### 6.3 Check System Pods

```bash
kubectl get pods -n kube-system
# All pods should be Running or Completed
```

### 6.4 Check Application Health

```bash
kubectl get pods --all-namespaces
# No pods should be in CrashLoopBackOff or Error state
```

### 6.5 Run Smoke Tests

```bash
# Test DNS resolution
kubectl run test-dns --image=busybox --restart=Never --rm -it -- nslookup kubernetes.default

# Test pod scheduling
kubectl run test-pod --image=nginx --restart=Never
kubectl get pod test-pod
kubectl delete pod test-pod
```

### 6.6 Verify Add-on Health

```bash
aws eks list-addons --cluster-name <cluster-name>

for addon in vpc-cni coredns kube-proxy; do
  aws eks describe-addon \
    --cluster-name <cluster-name> \
    --addon-name $addon \
    --query "addon.{Name:addonName, Status:status, Version:addonVersion}"
done
```

### 6.7 Review CloudWatch Metrics

Check the following metrics in CloudWatch for anomalies:

- `cluster_failed_node_count`
- `node_cpu_utilization`
- `node_memory_utilization`
- API server request error rates

---

## Rollback Plan

EKS **does not support downgrading** the control plane version. Prevention is the only rollback option for the control plane.

| Component | Rollback Option |
|-----------|----------------|
| Control Plane | ❌ Not supported — must upgrade forward |
| Add-ons | ✅ Downgrade via `aws eks update-addon` to previous version |
| Managed Node Groups | ✅ Re-specify an older AMI release version |
| Self-Managed Nodes | ✅ Revert Launch Template to previous AMI and recycle nodes |
| Workloads | ✅ Restore from Velero/backup or revert Helm/GitOps deployment |

### Preserve Old Nodes Before Upgrade (Recommended)

Before terminating drained old nodes, keep them cordoned (but running) until the new nodes pass validation. Only terminate old nodes once you are confident the upgrade is successful.

---

## Troubleshooting

### Nodes stuck in `NotReady`

```bash
kubectl describe node <node-name>
# Check for taints, missing CNI, or kubelet errors

# Restart kubelet on the node (via SSM)
aws ssm start-session --target <instance-id>
sudo systemctl restart kubelet
```

### Add-on stuck in `DEGRADED`

```bash
aws eks describe-addon --cluster-name <cluster-name> --addon-name <addon-name>
# Check "configurationSchema" and "health.issues" in output

kubectl describe pod -n kube-system -l k8s-app=<addon-label>
```

### Pods stuck in `Pending` after node upgrade

```bash
kubectl describe pod <pod-name>
# Check Events section for scheduling failures, resource limits, or node taints
```

### kubectl commands failing after upgrade

Ensure your `kubectl` client version is within ±1 minor version of the server:

```bash
kubectl version
# Update kubectl if needed
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

### API deprecation errors

```bash
# Check for deprecated API usage
kubectl api-resources
kubectl get events --all-namespaces | grep -i deprecated
```

---

## Quick Reference Checklist

```
Pre-Upgrade
  [ ] Reviewed Kubernetes and EKS release notes
  [ ] Checked deprecated API usage (pluto or kubectl-convert)
  [ ] Verified add-on compatibility
  [ ] Backed up cluster resources
  [ ] Notified stakeholders

Control Plane
  [ ] Triggered control plane upgrade
  [ ] Waited for status: ACTIVE
  [ ] Confirmed new server version with kubectl

Add-ons
  [ ] Upgraded vpc-cni
  [ ] Upgraded coredns
  [ ] Upgraded kube-proxy
  [ ] Upgraded other add-ons (ebs-csi, etc.)

Node Groups
  [ ] Upgraded managed node groups
  [ ] Cordoned, drained, and replaced self-managed nodes
  [ ] Verified all nodes are Ready at new version

Post-Upgrade
  [ ] Verified all system pods are healthy
  [ ] Verified all application pods are healthy
  [ ] Ran smoke tests
  [ ] Reviewed CloudWatch metrics
```

---

*Last updated: April 2026 | Compatible with EKS Kubernetes versions 1.27–1.30*
