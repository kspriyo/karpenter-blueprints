# Karpenter Blueprint: Update Nodes using Drift

## Purpose
After upgrading the Kubernetes control plane version, you might be wondering how to properly upgrade the data plane nodes launched by Karpenter. Currently, Karpenter has a feature gate to mark nodes as drifted. A drifted node is one whose spec and metadata does not match the spec of its `Provisioner` and `ProviderRef`. A node can drift when a user changes their `Provisioner` or `ProviderRef`. Moreover, underlying infrastructure in the provider can be changed outside of the cluster. For example, configuring an `AMISelector` to match the control plane version in the `AWSNodeTemplate`. This allows you to control when to upgrade node's version or when a new AL2 EKS Optimized AMI is released, creating drifted nodes.

Karpenter's drift will reconcile when a node's AMI drifts from provisioning requirements. When upgrading a node, Karpenter will minimize the downtime of the applications on the node by initiating provisioning logic for a replacement node before terminating drifted nodes. Once Karpenter has begun provisioning the replacement node, Karpenter will cordon and drain the old node, terminating it when it’s fully drained, then finishing the upgrade.

## Requirements

* A Kubernetes cluster with Karpenter installed. You can use the blueprint we've used to test this pattern at the `cluster` folder in the root of this repository.

## Deploy
Let's start by enabling the drift feature gate in Karpenter's configmap settings. To do so, run this command:

```
kubectl -n karpenter get cm karpenter-global-settings -o yaml | \
  sed -e 's|featureGates.driftEnabled: "false"|featureGates.driftEnabled: "true"|' | \
  kubectl apply -f -
```

You can confirm the configuration has been updated running the following command:

```
kubectl -n karpenter get cm karpenter-global-settings -o jsonpath='{.data}' | grep driftEnabled
```

Now, you need to restart Karpenter's pods, run this command:

```
kubectl rollout restart deployment karpenter -n karpenter
```

Once drift is enabled, let's create a new `AWSNodeTemplate` to be more precise about the AMIs you'd like to use. For now, you'll intentionally create new nodes using a previous EKS version to simulate where you'll be after upgrading the control plane. 

```
  amiSelector:
    aws::name: '*-1.26-*' # Will get the latest AMI of this Kubernetes version
    aws::owners: self,amazon
```

If you're using the Terraform template provided in this repo, run the following commands to get the EKS cluster name and the IAM Role name for the Karpenter nodes:

```
export CLUSTER_NAME=$(terraform -chdir="../../cluster/terraform" output -raw cluster_name)
export KARPENTER_NODE_IAM_INSTANCE_PROFILE_NAME=$(terraform -chdir="../../cluster/terraform" output -raw node_instance_profile_name)
```

***NOTE***: If you're not using Terraform, you need to get those values manually. `CLUSTER_NAME` is the name of your EKS cluster (not the ARN), and `KARPENTER_NODE_IAM_INSTANCE_PROFILE_NAME` is the [instance profile](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html#instance-profiles-manage-console), which is a way to pass a single IAM role to the EC2 instance launched by the Karpenter provisioner. Typically, the instance profile name is the same as the IAM role, but to avoid errors, go to the IAM Console and get the instance profile name assigned to the role (not the ARN).

Now, make sure you're in this blueprint folder, then run the following command to create the new `Provisioner` and `AWSNodeTemplate`:

```
sed -i "s/<<CLUSTER_NAME>>/$CLUSTER_NAME/g" latest-current-ami.yaml
sed -i "s/<<KARPENTER_NODE_IAM_INSTANCE_PROFILE_NAME>>/$KARPENTER_NODE_IAM_INSTANCE_PROFILE_NAME/g" latest-current-ami.yaml
kubectl apply -f .
```

## Results
The pods from the sample workload should be running even if the node has a version that doesn't match with the control plane.

```
> kubectl get pods
NAME                                  READY   STATUS    RESTARTS     AGE
latest-current-ami-5bbfbc98f7-6hxkw   1/1     Running   0            3m
latest-current-ami-5bbfbc98f7-n7mgs   1/1     Running   0            3m
latest-current-ami-5bbfbc98f7-rxjjx   1/1     Running   0            3m
```

You should see a new node registered with the latest AMI for EKS `v1.26`, like this:

```
> kubectl get nodes -l karpenter.sh/initialized=true
NAME                                        STATUS   ROLES    AGE     VERSION
ip-10-0-48-23.eu-west-1.compute.internal    Ready    <none>   6m28s   v1.26.7-eks-8ccc7ba
```

Let's simulate a node upgrade by changing the EKS version in the `AWSNodeTemplate`, run this command:

```
kubectl -n karpenter get awsnodetemplate latest-current-ami-template -o yaml | \
  sed -e 's|1.26|1.27|' | \
  kubectl apply -f -
```

You can confirm the update has been applied by running this command:

```
kubectl get awsnodetemplate latest-current-ami-template -o yaml
```

Wait around two minutes, in the mean time, you can monitor Karpenter logs until you see something like this:

```
2023-09-13T16:54:21.200Z	DEBUG	controller.machine.disruption	marking machine as drifted	{"commit": "34d50bf-dirty", "machine": "latest-current-ami-p25g5"}
2023-09-13T16:54:31.151Z	INFO	controller.deprovisioning	deprovisioning via drift replace, terminating 1 machines ip-10-0-48-23.eu-west-1.compute.internal/m5.xlarge/spot and replacing with machine from types c6in.xlarge, r6id.2xlarge, r7a.xlarge, m5dn.12xlarge, m7i.16xlarge and 261 other(s)	{"commit": "34d50bf-dirty"}
```

You should now see a new node with the latest AMI version that matches the control plane's version.

```
> kubectl get nodes -l karpenter.sh/initialized=true
NAME                                        STATUS   ROLES    AGE   VERSION
ip-10-0-60-27.eu-west-1.compute.internal    Ready    <none>   20s   v1.27.4-eks-8ccc7ba
```

You can repeat this process every time you need to run a controlled upgrade of the nodes.

## Cleanup
To remove all objects created, simply run the following commands:

```
kubectl delete -f .
```