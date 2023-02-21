export EKS_CLUSTER_NAME=eks-dev-cluster
export EBS_CSI_POLICY_NAME=AmazonEBSCSIPolicy
export AWS_REGION=eu-west-1

# associate-iam-oidc-provider
eksctl utils associate-iam-oidc-provider --region=eu-west-1 --cluster=eks-dev-cluster --approve

# Let's attach IAM role to Kubernetes service account using eksctl.
eksctl create iamserviceaccount \
  --name aws-node \
  --namespace kube-system \
  --cluster "eks-dev-cluster" \
  --role-name "AmazonEKSVPCCNIRole" \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy \
  --override-existing-serviceaccounts \
  --approve

# Before we add aws-ebs-csi-driver, we first need to create an IAM role, and associate it with Kubernetes service account.

Let's use an example policy file, which you can download using the command below.
curl -sSL -o ebs-csi-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json

# Create a new IAM policy with that file
export EBS_CSI_POLICY_NAME=AmazonEBSCSIPolicy
export AWS_REGION=eu-west-1

aws iam create-policy \
--region $AWS_REGION \
--policy-name $EBS_CSI_POLICY_NAME \
--policy-document file://ebs-csi-policy.json

export EBS_CSI_POLICY_ARN=$(aws --region $AWS_REGION iam list-policies --query 'Policies[?PolicyName==`'$EBS_CSI_POLICY_NAME'`].Arn' --output text)
echo $EBS_CSI_POLICY_ARN
# Output
{
    "Policy": {
        "PolicyName": "AmazonEBSCSIPolicy",
        "PolicyId": "ANPA46FB4FFSOG54AILD7",
        "Arn": "arn:aws:iam::889397717348:policy/AmazonEBSCSIPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2023-02-21T08:33:35+00:00",
        "UpdateDate": "2023-02-21T08:33:35+00:00"
    }
}
# Attach the new policy to Kubernetes service account.
export EBS_CSI_POLICY_ARN=arn:aws:iam::889397717348:policy/AmazonEBSCSIPolicy

eksctl create iamserviceaccount \
  --cluster $EKS_CLUSTER_NAME \
  --name ebs-csi-controller-irsa \
  --namespace kube-system \
  --attach-policy-arn $EBS_CSI_POLICY_ARN \
  --override-existing-serviceaccounts --approve

# Installing aws-ebs-csi-driver #
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

# Install aws-ebs-csi-driver
helm upgrade --install aws-ebs-csi-driver \
  --version=1.2.4 \
  --namespace kube-system \
  --set serviceAccount.controller.create=false \
  --set serviceAccount.snapshot.create=false \
  --set enableVolumeScheduling=true \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true \
  --set serviceAccount.snapshot.name=ebs-csi-controller-irsa \
  --set serviceAccount.controller.name=ebs-csi-controller-irsa \
  aws-ebs-csi-driver/aws-ebs-csi-driver

# Verify
kubectl get pod -n kube-system -l "app.kubernetes.io/name=aws-ebs-csi-driver,app.kubernetes.io/instance=aws-ebs-csi-driver"
# Output verification
NAME                                  READY   STATUS    RESTARTS   AGE
ebs-csi-controller-6c7dc55b8f-8wh7s   6/6     Running   0          26s
ebs-csi-controller-6c7dc55b8f-sbhzp   6/6     Running   0          26s
ebs-csi-node-4f59n                    3/3     Running   0          26s
ebs-csi-node-h9jpk                    3/3     Running   0          26s
ebs-snapshot-controller-0             1/1     Running   0          26s

# StorageClass
Create a storage class to specify the provisioner

# Create volume
aws ec2 create-volume --region eu-west-1 --availability-zone eu-west-1a --size 10 --volume-type gp3

