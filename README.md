# DSDL (DLTK) App and new EKS API.

## Instructions to perform either a fresh Install or perform an Upgrade

## :NOTE: Create a backup of your existing K8s/contianer environment.


## When in doubt, please reach out to your account team or dev team via fdse@splunk.com

# Sample Cluster used: dltk-cluster122
The cluster is running on west-1 with name dltk-cluster122
https://us-west-1.console.aws.amazon.com/eks/home?region=us-west-1#/clusters

# 1. Create Cluster
```
eksctl create cluster --name dltk-cluster122 --version 1.22 --region us-west-1 --nodegroup-name dltk-nodes122 --node-type t2.2xlarge --nodes 2
```

# 2. Install kubectl

**Working version: 1.22**
```
kubectl version
Client Version: version.Info{Major:"1", Minor:"22+", GitVersion:"v1.22.17-eks-49d8fe8", GitCommit:"21484dfcc2c4db50f96f56eb138afe01cfa7ea08", GitTreeState:"clean", BuildDate:"2023-01-11T03:27:38Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"22+", GitVersion:"v1.22.17-eks-e0e89e9", GitCommit:"4254aea90beb594abb9244689f9743ec2006e71c", GitTreeState:"clean", BuildDate:"2022-12-20T14:30:25Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
```

**Note**: getting error in VScode Kubenetes extension. Error message
```
error: exec plugin: invalid apiVersion "client.authentication.k8s.io/v1alpha1
```

### 2.1 Wrong way to install kubectl
- used `sudo snap install kubectl --classic` to install kubectl
- Got `error: exec plugin: invalid apiVersion "client.authentication.k8s.io/v1alpha1` error when use kubectl

### 2.2 Correct way to install kubectl
- follow steps [here](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) to install the kubectl for `Kubernetes 1.22`.
- this version fix the above apiVersion error
- BUT this error still remain in VScode Kubenetes extenstion.
- Verify kubectl works
```
kubectl get svc
```

### Useful commands in terminal
- View Kubernetes resources
```
kubectl get nodes -o wide
```

- View the workloads running on your cluster
```
kubectl get pods -A -o wide
```

# 3. Setup EFS CSI driver in EKS
## 3.1 Install the Amazon EFS driver

1. install helm.
**Working version: 3.8.2**
```

```
helm version
version.BuildInfo{Version:"v3.8.2", GitCommit:"6e3701edea09e5d55a8ca2aae03a68917630e91b", GitTreeState:"clean", GoVersion:"go1.17.5"}
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh

# install helm with specific version
./get_helm.sh -v v3.8.2
```
2. Add the Helm repo.
```
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
```
3. Update the repo.
```
helm repo update
```

`4. Install a release of the driver using the Helm chart. Replace the repository address with the cluster's container image address.
```
/usr/local/bin/helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.us-west-1.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
```

5. To verify that aws-efs-csi-driver has started, run:
```
kubectl get pod -n kube-system -l "app.kubernetes.io/name=aws-efs-csi-driver,app.kubernetes.io/instance=aws-efs-csi-driver"
```

## 3.2 Create an Amazon EFS file system

1. Retrieve the VPC ID that your cluster is in and store it in a variable for use in a later step. ~
```
vpc_id=$(aws eks describe-cluster \
    --name dltk-cluster122 \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --region us-west-1 \
    --output text)
```

2. Retrieve the CIDR range for your cluster's VPC and store it in a variable for use in a later step. Replace region-code with the AWS Region that your cluster is in.
```
cidr_range=$(aws ec2 describe-vpcs \
    --vpc-ids $vpc_id \
    --query "Vpcs[].CidrBlock" \
    --output text \
    --region us-west-1)
```

3. Create a security group with an inbound rule that allows inbound NFS traffic for your Amazon EFS mount points.
```
security_group_id=$(aws ec2 create-security-group \
    --group-name MyEfsSecurityGroup \
    --description "My EFS security group" \
    --vpc-id $vpc_id \
    --output text \
    --region us-west-1)
```

4. Create an inbound rule that allows inbound NFS traffic from the CIDR for your cluster's VPC.
```
aws ec2 authorize-security-group-ingress \
    --group-id $security_group_id \
    --protocol tcp \
    --port 2049 \
    --cidr $cidr_range \
    --region us-west-1
```

## 3.3 Create an Amazon EFS file system for your Amazon EKS cluster.
1. Create a file system.
```
file_system_id=$(aws efs create-file-system \
    --region us-west-1 \
    --performance-mode generalPurpose \
    --query 'FileSystemId' \
    --output text)
```

2. Create mount targets.
2.1 Determine the IP address of your cluster nodes.
```
kubectl get nodes
```

**output**
```
NAME                                          STATUS   ROLES    AGE    VERSION
ip-192-168-2-242.us-west-1.compute.internal   Ready    <none>   125m   v1.22.15-eks-fb459a0
ip-192-168-53-92.us-west-1.compute.internal   Ready    <none>   125m   v1.22.15-eks-fb459a0
```

2.2 Determine the IDs of the subnets in your VPC and which Availability Zone the subnet is in.
```
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$vpc_id" \
    --query 'Subnets[*].{SubnetId: SubnetId,AvailabilityZone: AvailabilityZone,CidrBlock: CidrBlock}' \
    --region us-west-1 \
    --output table
```

**output**
```
---------------------------------------------------------------------
|                          DescribeSubnets                          |
+------------------+-------------------+----------------------------+
| AvailabilityZone |     CidrBlock     |         SubnetId           |
+------------------+-------------------+----------------------------+
|  us-west-1c      |  192.168.32.0/19  |  subnet-0cc3da1ab6453731b  |
|  us-west-1c      |  192.168.96.0/19  |  subnet-09f940831a7e4020d  |
|  us-west-1b      |  192.168.64.0/19  |  subnet-025ab2e91cc9667ba  |
|  us-west-1b      |  192.168.0.0/19   |  subnet-0b5e458a9e15d5386  |
+------------------+-------------------+----------------------------+
```

2.3 Add mount targets for the subnets that your nodes are in.
# ip-192-168-2-242.us-west-1.compute.internal -> 192.168.0.0/19 
```
aws efs create-mount-target \
    --file-system-id $file_system_id \
    --subnet-id subnet-0b5e458a9e15d5386 \
    --security-groups $security_group_id \
    --region us-west-1
```

**output**
```
{
    "OwnerId": "158540654927",
    "MountTargetId": "fsmt-06371a8a56c98b519",
    "FileSystemId": "fs-0cc6268fe8b1992d1",
    "SubnetId": "subnet-0b5e458a9e15d5386",
    "LifeCycleState": "creating",
    "IpAddress": "192.168.5.83",
    "NetworkInterfaceId": "eni-033dee75108227fc8",
    "AvailabilityZoneId": "usw1-az3",
    "AvailabilityZoneName": "us-west-1b"
}
```

# ip-192-168-53-92.us-west-1.compute.internal ->  192.168.32.0/19
```
aws efs create-mount-target \
    --file-system-id $file_system_id \
    --subnet-id subnet-0cc3da1ab6453731b \
    --security-groups $security_group_id \
    --region us-west-1
```

**output**
```
    "OwnerId": "158540654927",
    "MountTargetId": "fsmt-0747b068bbc952511",
    "FileSystemId": "fs-0cc6268fe8b1992d1",
    "SubnetId": "subnet-0cc3da1ab6453731b",
    "LifeCycleState": "creating",
    "IpAddress": "192.168.57.66",
    "NetworkInterfaceId": "eni-0defa2e58a55b6c7f",
    "AvailabilityZoneId": "usw1-az1",
    "AvailabilityZoneName": "us-west-1c"
}
```

## 3.4 Deploy a sample app that dynamically creates a persistent volume
1. Create a storage class for EFS.

1.1 Retrieve your Amazon EFS file system ID. You can find this in the Amazon EFS console, or use the following AWS CLI command.
```
aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text
```

**output**
```
fs-0a20b27c5b223dc94	fs-0cc6268fe8b1992d1
```

1.2 Download a StorageClass manifest for Amazon EFS.
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml
```

1.3. Edit the file. Find the following line, and replace the value for fileSystemId with your file system ID.
```
fileSystemId: fs-0a20b27c5b223dc94
```

1.4 Deploy the storage class.
```
kubectl apply -f storageclass.yaml
```

2. Test automatic provisioning by deploying a Pod that makes use of the PersistentVolumeClaim

2.1 Download a manifest that deploys a pod and a PersistentVolumeClaim.
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/pod.yaml
```

2.2 Deploy the pod with a sample app and the PersistentVolumeClaim used by the pod.
```
kubectl apply -f pod.yaml
```

2.3 Determine the names of the pods running the controller.
```
kubectl get pods -n kube-system | grep efs-csi-controller
```

**output**
```
efs-csi-controller-569f755df-gdrkb    3/3     Running            0          22m
efs-csi-controller-569f755df-l5khb    3/3     Running            0          22m
efs-csi-controller-66b9ff9b48-wmb94   2/3     ImagePullBackOff   0          22m
```

2.4 After few seconds, you can observe the controller picking up the change (edited for readability), where `efs-csi-controller-569f755df-gdrkb` is one of the pods in your output from the previous command.
```
kubectl logs efs-csi-controller-569f755df-gdrkb \
    -n kube-system \
    -c csi-provisioner \
    --tail 10
```

**output**
```
...

I0201 20:24:30.811027       1 controller.go:1442] provision "default/efs-claim" class "efs-sc": volume "pvc-f48f7fd1-a8ac-405d-82b7-4cef41c405e7" provisioned
I0201 20:24:30.811039       1 controller.go:1455] provision "default/efs-claim" class "efs-sc": succeeded
I0201 20:24:30.822183       1 event.go:285] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"default", Name:"dltk", UID:"60cedefc-68e5-4642-9f74-16bad5bddb15", APIVersion:"v1", ResourceVersion:"178434", FieldPath:""}): type: 'Normal' reason: 'ProvisioningSucceeded' Successfully provisioned volume pvc-60cedefc-68e5-4642-9f74-16bad5bddb15
I0201 20:24:30.826521       1 event.go:285] Event(v1.ObjectReference{Kind:"PersistentVolumeClaim", Namespace:"default", Name:"efs-claim", UID:"f48f7fd1-a8ac-405d-82b7-4cef41c405e7", APIVersion:"v1", ResourceVersion:"175768", FieldPath:""}): type: 'Normal' reason: 'ProvisioningSucceeded' Successfully provisioned volume pvc-f48f7fd1-a8ac-405d-82b7-4cef41c405e7
```

2.5 Confirm that a persistent volume was created with a status of Bound to a PersistentVolumeClaim
```
kubectl get pv
```

**output**
```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pvc-60cedefc-68e5-4642-9f74-16bad5bddb15   1Gi        RWX            Delete           Bound    default/dltk        efs-sc                  20h
pvc-f48f7fd1-a8ac-405d-82b7-4cef41c405e7   5Gi        RWX            Delete           Bound    default/efs-claim   efs-sc                  20h
```

2.6 View details about the PersistentVolumeClaim that was created.
```
kubectl get pvc
```

**output**
```
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
dltk        Bound    pvc-60cedefc-68e5-4642-9f74-16bad5bddb15   1Gi        RWX            efs-sc         2d4h
efs-claim   Bound    pvc-f48f7fd1-a8ac-405d-82b7-4cef41c405e7   5Gi        RWX            efs-sc         2d4h
```

2.7 View the sample app pod's status until the STATUS becomes Running.
```
kubectl get pods -o wide
```

**output**
```
NAME                   READY   STATUS              RESTARTS   AGE    IP               NODE                                          NOMINATED NODE   READINESS GATES
dev-86dc947455-qcw6c   0/1     ContainerCreating   0          20h    <none>           ip-192-168-53-92.us-west-1.compute.internal   <none>           <none>
efs-app                1/1     Running             0          2d4h   192.168.24.207   ip-192-168-2-242.us-west-1.compute.internal   <none>           <none>
```

2.8 Confirm that the data is written to the volume.
```
kubectl exec efs-app -- bash -c "cat data/out"
```

**output**
```
...

Thu Feb 2 17:21:35 UTC 2023
Thu Feb 2 17:21:40 UTC 2023
Thu Feb 2 17:21:45 UTC 2023
Thu Feb 2 17:21:50 UTC 2023
Thu Feb 2 17:21:55 UTC 2023
Thu Feb 2 17:22:00 UTC 2023
Thu Feb 2 17:22:05 UTC 2023
```

3.0 Validate the example use-case in DSDL (DLTK) App




# For help, please create an issue or reach out to us via fdse@splunk.com






