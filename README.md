# EKS-cluster

# 2.1 Create AWS IAM user, access key, and EKS cluster IAM role from Console

Create a free account first
```
https://aws.amazon.com/resources/create-account/
```
Instead of using `default` profile created by above `aws configure`, you can create a named AWS Profile `eks-demo` in two ways:

1. `aws configure --profile eks-demo`
2. create profile entry in `~/.aws/credentials` file

To create profile entry in `~/.aws/credentials` file, do the followings:
```
vim ~/.aws/credentials
```
Enter `i` key and paste below lines into the file
```
[eks-demo] 
aws_access_key_id=YOUR_ACCESS_KEY 
aws_secret_access_key=YOUR_SECRET_ACCESS_KEY
aws_region = YOUR_REGION 
```

Then check if new profile can be authenticated
```sh
export AWS_PROFILE=eks-demo

```
aws sts get-caller-identity
```



# 2.3 Install aws-iam-authenticator (if aws cli is 1.20.156 or earlier)
```bash
# Mac
brew install aws-iam-authenticator

# Windows
# install chocolatey first: https://chocolatey.org/install
choco install -y aws-iam-authenticator
```


# 2.5 Install eksctl
Ref: https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html
```bash
# Mac
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
eksctl version

# Windows: https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

# install eskctl from chocolatey
chocolatey install -y eksctl 

eksctl version

# Windows: https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html
# install chocolatey first
https://chocolatey.org/install

# instakk eskctl from chocolatey
chocolatey install -y eksctl 
```


# 2.6 Create ssh key for EKS worker nodes
```bash
ssh-keygen
eks_demo.pem
```

# 2.7 Setup EKS cluster with eksctl (so you don't need to manually create VPC)
`eksctl` tool will create K8s Control Plane (master nodes, etcd, API server, etc), worker nodes, VPC, Security Groups, Subnets, Routes, Internet Gateway, etc.
```bash
# use official AWS EKS AMI
# dedicated VPC
# EKS not supported in us-west-1

eksctl create cluster \
    --name eks-from-eksctl \
    --version 1.21 \
    --region ap-southeast-1 \
    --nodegroup-name workers \
    --node-type t2.small \
    --nodes 2 \
    --nodes-min 1 \
    --nodes-max 4 \
    --ssh-access \
    --ssh-public-key ~/.ssh/eks-demo.pub \
    --managed
```



Once you have created a cluster, you will find that cluster credentials were added in ~/.kube/config

```bash
# get info about cluster resources
aws eks describe-cluster --name eks-from-eksctl --region ap-southeast-1
```

```output
2021-11-29 15:21:55 [ℹ]  waiting for the control plane availability...
2021-11-29 15:21:55 [✔]  saved kubeconfig as "/Users/rupeshpanwar/.kube/config"
2021-11-29 15:21:55 [ℹ]  no tasks
2021-11-29 15:21:55 [✔]  all EKS cluster resources for "eks-from-eksctl" have been created
2021-11-29 15:21:55 [ℹ]  nodegroup "workers" has 2 node(s)
2021-11-29 15:21:55 [ℹ]  node "ip-192-168-27-88.ap-southeast-1.compute.internal" is ready
2021-11-29 15:21:55 [ℹ]  node "ip-192-168-61-64.ap-southeast-1.compute.internal" is ready
2021-11-29 15:21:55 [ℹ]  waiting for at least 1 node(s) to become ready in "workers"
2021-11-29 15:21:56 [ℹ]  nodegroup "workers" has 2 node(s)
2021-11-29 15:21:56 [ℹ]  node "ip-192-168-27-88.ap-southeast-1.compute.internal" is ready
2021-11-29 15:21:56 [ℹ]  node "ip-192-168-61-64.ap-southeast-1.compute.internal" is ready
2021-11-29 15:21:57 [ℹ]  kubectl command should work with "/Users/rupeshpanwar/.kube/config", try 'kubectl get nodes'
2021-11-29 15:21:57 [✔]  EKS cluster "eks-from-eksctl" in "ap-southeast-1" region is ready
```

```bash
# get info about cluster resources
aws eks describe-cluster --name eks-from-eksctl --region ap-southeast-1
```

# 3. Helm Chart Quick Intro

## Install Helm v3 (tiller is deprecated)
Ref: https://helm.sh/docs/intro/install/
```bash
brew install helm

# Windows
choco install kubernetes-helm

helm version
```

## Helm Chart Anatomy
```bash
$ tree

.
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```
## Helm Commands
Add some of popular Helm repo
```bash
# deprecated
# helm repo add stable https://kubernetes-charts.storage.googleapis.com/

# new chart link
https://charts.helm.sh/stable
```

Search charts in repo
```bash
helm search repo stable

# this is nginx for ingress controller
helm search repo nginx

# for standalone nginx, add bitnami repo
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install my-release bitnami/nginx
helm repo update

#　search again
helm search repo nginx

# now bitnami/nginx shows up
helm search repo bitnami/nginx
```


Install
```bash
# USAGE: helm install RELEASE_NAME REPO_NAME
helm install nginx bitnami/nginx
```


Upgdate
```bash
# upgrade after changing values in yaml
helm upgrade nginx bitnami/nginx --dry-run

# upgrade using values in overrides.yaml
helm upgrade nignx bitnami/nginx -f overrides.yaml

# rollback
helm rollback nginx REVISION_NUMBER
```

Other commands
```bash
helm list 
helm status nginx
helm history nginx

# get manifest and values from deployment
helm get manifest nginx
helm get values nginx

helm uninstall nginx
```
# 4. Deploy Kuberenetes Dashboard
# 4.1 Required setup 1: Install Metrics Server first so Dashboard can poll metrics
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

Get token (kinda like password) for dashboard
```
kubectl describe secret $(k get secret -n kubernetes-dashboard | grep kubernetes-dashboard-token | awk '{ print $1 }') -n kubernetes-dashboard
```

Create a secure channel from local to API server in Kubernetes cluster
```
kubectl proxy

# access this url from browser
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

# 4.3 Required setup 3: Create RBAC to control what metrics can be visible

[eks-admin-service-account.yaml](eks-admin-service-account.yaml)
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin # this is the cluster admin role
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
```

Apply
```
kubectl apply -f eks-admin-service-account.yaml
```

Check it created in `kube-system` namespace
```
kubectl get serviceaccount -n kube-system | grep eks-admin
eks-admin                            1         52s
```

Get a token from the `eks-admin` serviceaccount
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

Create a secure channel from local to API server in Kubernetes cluster
```
kubectl proxy

# access this url from browser
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
# 4.4 K8s Dashboard Walkthrough


## Uninstall Dashboard
```
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml

kubectl delete -f eks-admin-service-account.yaml
```




