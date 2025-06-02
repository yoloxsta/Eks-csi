# Managing Persistent Storage with Amazon EFS on EKS cluster

1. Download the IAM policy:
   ```bash
   curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json
   ```
2. Create an IAM policy:
   ```bash
   aws iam create-policy \
       --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
       --policy-document file://iam-policy-example.json
   ```
3. To determine your cluster's OIDC provider ID, run the following command:
   ```bash
   aws eks describe-cluster --name your_cluster_name --query "cluster.identity.oidc.issuer" --output text
   ```
   _**Note:**_ Replace _**your_cluster_name**_ with your cluster name.
4. Create the following IAM trust policy, and then grant the AssumeRoleWithWebIdentity action to your Kubernetes service account:
   ```json
   cat <<EOF > trust-policy.json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "oidc.eks.YOUR_AWS_REGION.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>:sub": "system:serviceaccount:kube-system:efs-csi-controller-sa"
           }
         }
       }
     ]
   }
   EOF
   ```
   _**Note:**_ Replace _**YOUR_AWS_ACCOUNT_ID**_ with your account ID, _**YOUR_AWS_REGION**_ with your AWS Region, and _**XXXXXXXXXX45D83924220DC4815XXXXX**_ with your cluster's OIDC provider ID.

5. Create an IAM role:
   ```bash
   aws iam create-role \
       --role-name AmazonEKS_EFS_CSI_DriverRole \
       --assume-role-policy-document file://"trust-policy.json"
   ```
6. Attach your new IAM policy to the role:
   ```bash
   aws iam attach-role-policy \
       --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AmazonEKS_EFS_CSI_Driver_Policy \
       --role-name AmazonEKS_EFS_CSI_DriverRole
   ```
8. Save the following contents to a file that's named _efs-service-account.yaml_:
   ```bash
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     labels:
       app.kubernetes.io/name: aws-efs-csi-driver
     name: efs-csi-controller-sa
     namespace: kube-system
     annotations:
       eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/AmazonEKS_EFS_CSI_DriverRole
   ```
9. Create the Kubernetes service account on your cluster:
   ```bash
   kubectl apply -f efs-service-account.yaml
   ```
   _**Note:**_ The Kubernetes service account that's named _efs-csi-controller-sa_ is annotated with the IAM role that you created.
10. Download the manifest from the public Amazon ECR registry and use the images to install the driver:
    ```bash
    kubectl kustomize "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.5" > public-ecr-driver.yaml
    ```
    _**Note:**_ To install the EFS CSI driver, you can use Helm and a Kustomize with AWS Private or Public Registry. For instructions on how to install the EFS CSI Driver, see the AWS EFS CSI driver on the GitHub website.

11. Edit the file _public-ecr-driver.yaml_ and annotate _efs-csi-controller-sa_ Kubernetes service account section with the IAM role's ARN:
    ```bash
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      labels:
        app.kubernetes.io/name: aws-efs-csi-driver
      annotations:
        eks.amazonaws.com/role-arn: arn:aws:iam::<accountid>:role/AmazonEKS_EFS_CSI_DriverRole
      name: efs-csi-controller-sa
      namespace: kube-system
    ```
12. Deploy the Amazon EFS CSI driver:
    ```bash
    kubectl apply -f public-ecr-driver.yaml
    ```
### **[ Helm ]**
This procedure requires Helm V3 or later. To install or upgrade Helm, see [Using Helm with Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/helm.html).  
**To install the driver using Helm**
  1. Add the Helm repo:
     ```bash
     helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
     ```
  2. Update the repo:
        ```bash
        helm repo update aws-efs-csi-driver
        ```
  3. Install a release of the driver using the Helm chart:
     ```bash
     helm upgrade --install aws-efs-csi-driver --namespace kube-system aws-efs-csi-driver/aws-efs-csi-driver
     ```
### **Create an Amazon EFS File system**
   1. To get the virtual private cloud (VPC) ID of your Amazon EKS cluster, run the following command:
      ```bash
      aws eks describe-cluster --name your_cluster_name --query "cluster.resourcesVpcConfig.vpcId" --output text
      ```
      _**Note:**_ Replace *your_cluster_name* with your cluster name.
   2. To get the CIDR range for your VPC cluster, run the following command:
      ```bash
      aws ec2 describe-vpcs --vpc-ids YOUR_VPC_ID --query "Vpcs[].CidrBlock" --output text
      ```
      _**Note:**_ Replace the _**YOUR_VPC_ID**_ with your VPC ID.
   3. Create a security group that allows inbound network file system (NFS) traffic for your Amazon EFS mount points:
      ```bash
      aws ec2 create-security-group --description efs-test-sg --group-name efs-sg --vpc-id YOUR_VPC_ID
      ```
      _**Note:**_ Replace _**YOUR_VPC_ID**_ with your VPC ID. Note the _**GroupId**_ to use later.
   4. To allow resources in your VPC to communicate with your Amazon EFS file system, add an NFS inbound rule:
      ```bash
      aws ec2 authorize-security-group-ingress --group-id sg-xxx --protocol tcp --port 2049 --cidr YOUR_VPC_CIDR
      ```
      _**Note:**_ Replace _**YOUR_VPC_CIDR**_ with your VPC CIDR and _**sg-xxx**_ with your security group ID.
   5. Create an Amazon EFS file system for your Amazon EKS cluster:
      ```bash
      aws efs create-file-system --creation-token eks-efs
      ```
      _**Note:**_ Note the _**FileSystemId**_ to use later.
   6. To create a mount target for Amazon EFS, run the following command:
      ```bash
      aws efs create-mount-target --file-system-id FileSystemId --subnet-id SubnetID --security-group sg-xxx
      ```
      _**Important:**_ Run the preceding command for all the Availability Zones with the _**SubnetID**_ in the Availability Zone where your worker nodes are running. Replace _**FileSystemId**_ with your EFS file system's ID, _**sg-xxx**_ with your security group's ID, and _**SubnetID**_ with your worker node subnet's ID. To create mount targets in multiple subnets, run the command for each subnet ID. It's a best practice to create a mount target in each Availability Zone where your worker nodes are running. You can create mount targets for all the Availability Zones where worker nodes are launched. Then, all the Amazon Elastic Compute Cloud (Amazon EC2) instances in these Availability Zones can use the file system.
### **Test the Amazon EFS CSI driver**
  ### Multiple Pods Read Write Many 
      This example shows how to create a static provisioned Amazon EFS persistent volume (PV) and access it from multiple pods with the `ReadWriteMany` (RWX) access mode. This mode allows the PV to be read and written from multiple pods.

   1. Clone the [Amazon EFS Container Storage Interface \(CSI\) driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver) GitHub repository to your local system.

         ```sh
         git clone https://github.com/kubernetes-sigs/aws-efs-csi-driver.git
         ```

   1. Navigate to the `multiple_pods` example directory.

      ```sh
      cd aws-efs-csi-driver/examples/kubernetes/multiple_pods/
      ```

   1. Retrieve your Amazon EFS file system ID. You can find this in the Amazon EFS console, or use the following AWS CLI command.

      ```sh
      aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text
      ```

      The example output is as follows.

      ```
      fs-582a03f3
      ```

  1. Edit the [`specs/pv.yaml`](./specs/pv.yaml) file and replace the `volumeHandle` value with your Amazon EFS file system ID.

     ```
     apiVersion: v1
     kind: PersistentVolume
     metadata:
       name: efs-pv
     spec:
       capacity:
         storage: 5Gi
       volumeMode: Filesystem
       accessModes:
         - ReadWriteMany
       persistentVolumeReclaimPolicy: Retain
       storageClassName: efs-sc
       csi:
         driver: efs.csi.aws.com
         volumeHandle: fs-582a03f3
     ```
  **Note**  
    `spec.capacity` is ignored by the Amazon EFS CSI driver because Amazon EFS is an elastic file system. The actual storage capacity value in persistent volumes and persistent volume claims isn't used when creating the file system. However, because storage capacity is a required field in Kubernetes, you must specify a valid value, such as, `5Gi` in this example. This value doesn't limit the size of your Amazon EFS file system.



  1. Deploy the `efs-sc` storage class, the `efs-claim` PVC, and the `efs-pv` PV from the `specs` directory.

     ```sh
     kubectl apply -f specs/pv.yaml
     kubectl apply -f specs/claim.yaml
     kubectl apply -f specs/storageclass.yaml
     ```

  1. List the persistent volumes in the default namespace. Look for a persistent volume with the `default/efs-claim` claim.

     ```sh
     kubectl get pv -w
     ```

     The example output is as follows.

     ```
     NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
     efs-pv   5Gi        RWX            Retain           Bound    default/efs-claim   efs-sc                  2m50s
     ```

    Don't proceed to the next step until the `STATUS` is `Bound`.

  1. Deploy the `app1` and `app2` sample applications from the `specs` directory. Both `pod1` and `pod2` consume the PV and write to the same Amazon EFS filesystem at the same time.

     ```sh
     kubectl apply -f specs/pod1.yaml
     kubectl apply -f specs/pod2.yaml
     ```

  1. Watch the Pods in the default namespace and wait for the `app1` and `app2` Pods' `STATUS` to become `Running`.

     ```sh
     kubectl get pods --watch
     ```
  **Note**  
    It may take a few minutes for the Pods to reach the `Running` status.

  1. Describe the persistent volume.

     ```sh
     kubectl describe pv efs-pv
     ```

     The example output is as follows.

     ```
     Name:            efs-pv
     Labels:          none
     Annotations:     kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"name":"efs-pv"},"spec":{"accessModes":["ReadWriteMany"],"capaci...
                     pv.kubernetes.io/bound-by-controller: yes
     Finalizers:      [kubernetes.io/pv-protection]
     StorageClass:    efs-sc
     Status:          Bound
     Claim:           default/efs-claim
     Reclaim Policy:  Retain
     Access Modes:    RWX
     VolumeMode:      Filesystem
     Capacity:        5Gi
     Node Affinity:   none
     Message:
     Source:
         Type:              CSI (a Container Storage Interface (CSI) volume source)
         Driver:            efs.csi.aws.com
         VolumeHandle:      fs-582a03f3
         ReadOnly:          false
         VolumeAttributes:  none
     Events:                none
     ```

     The Amazon EFS file system ID is listed as the `VolumeHandle`.

  1. Verify that the `app1` Pod is successfully writing data to the volume.

     ```sh
     kubectl exec -ti app1 -- tail -f /data/out1.txt
     ```

     The example output is as follows.

     ```
     [...]
     Mon Mar 22 18:18:22 UTC 2021
     Mon Mar 22 18:18:27 UTC 2021
     Mon Mar 22 18:18:32 UTC 2021
     Mon Mar 22 18:18:37 UTC 2021
     [...]
     ```

  1. Verify that the `app2` Pod shows the same data in the volume that `app1` wrote to the volume.

     ```sh
     kubectl exec -ti app2 -- tail -f /data/out2.txt
     ```

     The example output is as follows.

     ```
     [...]
     Mon Mar 22 18:18:22 UTC 2021
     Mon Mar 22 18:18:27 UTC 2021
     Mon Mar 22 18:18:32 UTC 2021
     Mon Mar 22 18:18:37 UTC 2021
     [...]
     ```

  1. When you finish experimenting, delete the resources for this sample application to clean up.

     ```sh
     kubectl delete -f specs/
     ```

     You can also manually delete the file system and security group that you created.