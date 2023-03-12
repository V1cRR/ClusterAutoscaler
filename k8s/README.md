# Setup cluster-autoscaler for EKS cluster

Refference links
• Tagg your Amazon EC2 resources https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html
	 
• Create an IAM OIDC provider https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html
	 
• Install eksctl https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html
	 
• Create an Amazon EKS Cluster https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html


## Prerequisits:

	• Existing Amazon EKS Cluster
	 
	• IAM OIDC identity provider for the cluster with the AWS Management Console
	 
	• Node groups with Auto Scaling groups tags. 
	  The Cluster Autoscaler requires the following tags on your Auto Scaling groups so that they can be auto-discovered. 
	  Check your tags, if they are not there just add manually. 
	  If you used eksctl to create your node groups, these tags are automatically applied.
		Key	                                       Value
		
		k8s.io/cluster-autoscaler/my-cluster-name	owned
		k8s.io/cluster-autoscaler/enabled	    	true
        	eks.amazonaws.com/capacityType          	SPOT

### Create an IAM policy
    
    
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/k8s.io/cluster-autoscaler/ubuntu-eks-dev": "owned"
                }
            }
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeAutoScalingGroups",
                "ec2:DescribeLaunchTemplateVersions",
                "autoscaling:DescribeTags",
                "autoscaling:DescribeLaunchConfigurations"
            ],
            "Resource": "*"
        }
    ]
}
```
     
     


### Create an IAM role and attach an IAM policy to it

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::401413892014:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/77EFC96D1BBB3CF324FA2D1AB8910290"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"oidc.eks.us-east-1.amazonaws.com/id/77EFC96D1BBB3CF324FA2D1AB8910290:sub": "system:serviceaccount:kube-system:cluster-autoscaler"
				}
			}
		}
	]
}
```

  
  
  
##  Deploy the Cluster Autoscaler

1. Create the Cluster Autoscaler YAML file cluster-autoscaler.yaml 
    
    You can also use github repository and make some adjustments based on your cluster.
    
   ``` curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml```

2. Apply the YAML file to your cluster.
    kubectl apply -f cluster-autoscaler.yaml
    
3. Annotate the cluster-autoscaler service account with the ARN of the IAM role that you created previously.

4. Patch the deployment to add the cluster-autoscaler.kubernetes.io/safe-to-evict annotation to the Cluster Autoscaler pods with the following command:
    kubectl patch deployment cluster-autoscaler \
     -n kube-system \
     -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'

5. Edit the Cluster Autoscaler yaml file so it must have the following lines:
```
          command: 
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/ubuntu-eks-dev # Update cluster with your cluster name
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
```
     

6. Set the Cluster Autoscaler image to the latest available by checking the oficial repo in GitHub

	https://github.com/kubernetes/autoscaler/releases

7. View your Cluster Autoscaler logs with the following command.

     kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler

8. Create an NGINX.yaml.file. Make sure that under Spec your 
```
matchExpressions: 
- key: eks.amazonaws.com/capacityType
  operator: In
  values:
  - SPOT
```
Key and values must match with your AutoScaleGroup tag
    
9. Apply Nginx.yaml, check the pods, check the nodes after 2-3 minute should scale-out

    
# Issue that I faced
In the first documentation I found, they said that the Cluster Autoscaler image version must match the cluster version.Due to this, the deployment of the Cluster Autoscaler was pending.After a brief search on the official AWS page, I saw that they suggested using the latest version, checking the github repo. I readjusted my manifest file and the Cluster Autoscaler was created correctly.
