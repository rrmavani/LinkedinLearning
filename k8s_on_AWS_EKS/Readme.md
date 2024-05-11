Course : Linkedin Learning - running-kubernetes-on-aws-eks
INSTRUCTOR : Kim Schlesinger - Technical Curriculum Developer
Github repo : github.com:kimschles/k8s-on-aws.git 

.envrc
```
export AWS_PROFILE=linkedin-learning
```
.aws/config
```
[profile linkedin-learning]
region = us-west-2
output = json
cli_pager =
```

.aws/credentials
```
[linkedin-learning]
aws_access_key_id = xxxxxxxx
aws_secret_access_key = xxxxxxxxxx
````

https://leadbizru.signin.aws.amazon.com/console
username : rrmavani


### Some useful command
```
➜  k8s_on_AWS_EKS git:(main) aws eks list-clusters
{
    "clusters": []
}
➜  k8s_on_AWS_EKS git:(main) ✗ aws iam get-user     
{
    "User": {
        "Path": "/",
        "UserName": "xxxxxx",
        "UserId": "xxxxxxx",
        "Arn": "arn:aws:iam::xxxxxxxx:user/rrmavani",
        "CreateDate": "2024-05-08T19:20:11+00:00",
        "PasswordLastUsed": "2024-05-09T12:31:53+00:00",
        "Tags": [
            {
                "Key": "xxxxxxxxx",
                "Value": "eks-lil"
            },
            {
                "Key": "lil-eks",
                "Value": "lil-eks"
            }
        ]
    }
}
```
```
### Other commands
```
kubectl config get-contexts

eksctl create cluster -f cluster.yaml


kubectl get pods -A
kubectl get nodes
```

### Deploy application
```
kubectl apply -f namespace.yaml
kubectl get ns
kubectl apply -f deployment.yaml
kubectl get pods -n development
kubectl apply -f service.yaml
kubectl get services -n development
```

### Tag VPC subnets
- The AWS Load Balancer Controller must be able to identify the subnets in your cluster's VPC and it looks for them by querying for specific tags.  
  
Private Subnet Tags
```
kubernetes.io/cluster/lil-eks: shared
kubernetes.io/role/internal-elb: 1
```

Public Subnets Tags
```
kubernetes.io/cluster/lil-eks: shared
kubernetes.io/role/elb: 1
```

### Create a service account for the Load Balancer Controller
- Create an AWS IAM policy that is bound to a Kubernetes service account. You will create an IAM policy that has permissions to create an AWS elastic load balancer. 
- Attach this policy to a kubernetes role to which pods and Kubernetes have these permissions. 
- To use AWS IAM policy for k8s service accounts, an IAM OIDC provider must exist for your cluster's OIDC issuer URL.  


##### Create policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
#### Create OIDC provider
```
eksctl utils associate-iam-oidc-provider --cluster lil-eks --approve
```  
#### Create k8s service account and attach policy  
```
eksctl create iamserviceaccount \
    --cluster=lil-eks \
    --name=aws-load-balancer-controller \
    --namespace=kube-system \
    --attach-policy-arn=arn:aws:iam::396343719918:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve

kubectl get sa -n kube-system | grep aws-load-balancer-controller
```

#### Install the AWS Load Balancer Controller
- The AWS Load Balancer Controller manages elb for a k8s cluster. 
- In Kubernetes, a controller is a process that runs on a loop, checking for changes and reacting as needed. - The load balancer controller watches for any ingress events from the Kubernetes API server. When it sees any ingress resources that satisfy its requirements, it kicks off the creation of AWS resources. 
- The first thing to do is install Cert Manager. Cert Manager is a Kubernetes add-on that simplifies the creation and management of TLS certificates, which are required to enable https. 
- The AWS Load Balancer Controller needs Cert Manager to encrypt the traffic in and out of your cluster.

##### Install cert manager
```
kubectl apply \
    --validate=false \
    -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml

kubectl get pods -n cert-manager
```
##### Install Loadbalancer Controller
```
kubectl apply -f load-balancer-controller.yaml 

kubectl get deployment -n kube-system
```

#### Install Ingress
- 'IngressClass' and 'IngressClassParams' resources are required to connect with your 'Application Ingress' by 'Loadbalancer Controller'.  
```
kubectl apply -f ingress-class.yaml
```
- 'Application Ingress' is required to route traffic to application
```
kubectl apply -f pod-info-ingress.yaml 
```
Faced one issue related to access of k8s sa to add tags
solved by modifying the policy.json
Ref : https://github.com/kubernetes-sigs/aws-load-balancer-controller/issues/3399


#### Delete
Delete Loadbalancers
```

eksctl delete cluster --name lil-eks --disable-nodegroup-eviction


```
Verify below resources
VPC
EC2
CLOUD FORMATION
EKS
IAM POLICY
