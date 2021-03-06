# 7. Security: User Authentication (aws-iam-authenticator) and Authorization (RBAC: Role Based Access Control)

<img width="620" alt="image" src="https://user-images.githubusercontent.com/75510135/144391312-53b0e8ae-6532-4781-aec9-6aed21e763dd.png">

<img width="754" alt="image" src="https://user-images.githubusercontent.com/75510135/144392166-5ed8239f-04bb-4c39-9215-0c3a981258c2.png">

<img width="528" alt="image" src="https://user-images.githubusercontent.com/75510135/144392386-56757282-ce5e-4894-a5c8-a3e27659e464.png">

<img width="561" alt="image" src="https://user-images.githubusercontent.com/75510135/144392454-6c12c2d7-ea8f-44db-a87e-55addb75ed2e.png">
<img width="412" alt="image" src="https://user-images.githubusercontent.com/75510135/144392816-b6a73f43-94ca-4bc9-9322-f2b43600caa9.png">
<img width="441" alt="image" src="https://user-images.githubusercontent.com/75510135/144404097-ef91fc96-d957-44a1-bf71-400c867b7875.png">
# 7.1 AWS IAM User Authentication to K8s Cluster Process Breakdown
1. `kubectl ...` command sends a request to API server in K8s master node. `kubectl` will by default send authentication token stored in `~/.kube/config` (for AWS EKS, the token is AWS IAM user ARN or IAM Role ARN)
2. API server will pass along the token to `aws-iam-authenticator` server in the cluster, which in turn will ask AWS IAM about the user's identity
3. Once `aws-iam-authenticator` receives a response from AWS IAM, it'll check `aws-auth` configmap in `kube-system` namespace to see if the verified AWS IAM user is bound to any k8s roles 
4. API servier will approve and execute actions, or decline (“You must be logged in to the server (Unauthorized)”) response to kubectl client


# 7.2 Kubeconfig and aws-auth ConfigMap
```
kubectl config view
```

Output
```
users:
- name: 1592048816494086000@eks-from-eksctl.us-west-2.eksctl.io
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      args:
      - token
      - -i
      - eks-from-eksctl
      command: aws-iam-authenticator
      env:
      - name: AWS_STS_REGIONAL_ENDPOINTS
        value: regional
      - name: AWS_DEFAULT_REGION
        value: us-west-2
```

you can see that 
1. `rolearn: arn:aws:iam::xxxxxxxx:role/eksctl-eks-from-eksctl-nodegroup-NodeInstanceRole-R3EFEQC9U6U` has new k8s username `system:node:{{EC2PrivateDNSName}}` 
2. and it is added into list of groups `system:bootstrappers` and `system:nodes`.

# 7.3 Create New AWS IAM User

1. Create new AWS IAM user from IAM console with AWS access key and secret access key

2. Add named AWS profile to `~/.aws/credentials` and save AWS access key and secret access key
```
vim ~/.aws/credentials
```

```
[eks-viewer]
aws_access_key_id = REDUCTED
aws_secret_access_key = REDUCTED
region = ap-southeast-1

Check each user's access
```
export AWS_PROFILE=eks-viewer
aws sts get-caller-identity
```


# 7.4 Allow AWS IAM Users to K8s Cluster with Root Access (Don't do this!)

Refs: 
- https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles

```
kubectl edit -n kube-system configmap/aws-auth
```

Add this section to `aws-auth` configmap to add new AWS IAM user
```yaml
mapUsers: |
    - userarn: arn:aws:iam::111122223333:user/eks-viewer   # AWS IAM user
      username: this-aws-iam-user-name-will-have-root-access
      groups:
      - system:masters  # K8s User Group
```

Check edited content is correct by printing out yaml 
```bash
$ kubectl get -n kube-system configmap/aws-auth -o yaml


Check if AWS IAM user `eks-viewer` can now view K8s cluster
```sh
# switch AWS IAM user by changing AWS profile
export AWS_PROFILE=eks-viewer

# do some READ/GET operations with kubectl
kubectl get pod
NAME                 READY   STATUS    RESTARTS   AGE
guestbook-dxkpd      1/1     Running   0          4h6m
guestbook-fsqx8      1/1     Running   0          4h6m
guestbook-nnrjc      1/1     Running   0          4h6m
redis-master-6dbj4   1/1     Running   0          4h8m
redis-slave-c6wtv    1/1     Running   0          4h7m
redis-slave-qccp6    1/1     Running   0          4h7m
```
Check authorization of this AWS IAM user `eks-viewer`, bound to K8s user group `system:masters`
```
$ kubectl auth can-i create deployments
yes

$ kubectl auth can-i delete deployments
yes

$ kubectl auth can-i delete ClusterRole
yes

But __this is not recommended__ because `system:masters` has admin access to K8s resources (not AWS resources)
```sh
# check "cluster-admin" ClusterRoleBinding
$ kubectl get clusterrolebindings/cluster-admin -o yaml
```

This breaks the principle of least priviledge.

To assoicate AWS IAM user to right K8s user group, create K8s ClusterRoleBinding.

# 7.5 Restrict K8s User Access by Creating ClusterRoleBinding (RBAC - Role Based Access Control)

Refs: 
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles
- https://github.com/kubernetes-sigs/aws-iam-authenticator/issues/139#issuecomment-417400851

There are some default K8s ClusterRole in `kube-system`, such as `edit`, `view`, etc

Get a whole list
```
kubectl get clusterrole -n kube-system
````

Show details of permissions for `edit` ClusterRole
```
kubectl describe clusterrole edit -n kube-system
```
This clusterrole isn't bound to any K8s user __yet__. To attach this `edit` ClusterRole to a new K8s user `system:editor`, create `ClusterRoleBinding`.

Generate yaml file 
```sh
# bind ClusterRole "edit" to a new K8s user group "system:editor" by creating ClusterRoleBinding
kubectl create clusterrolebinding system:editor \
    --clusterrole=edit \
    --group=system:editor \
    --dry-run -o yaml > clusterrolebinding_system_editor.yaml

cat clusterrolebinding_system_editor.yaml
```
Output
```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: null
  name: system:editor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole  
  name: edit  # K8s ClusterRole name
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:editor  # K8s User Group
``` 

Apply
```
kubectl apply -f clusterrolebinding_system_editor.yaml
```

Check ClusterRoleBinding `system:editor` is bound to ClusterRole `edit`
```
$ kubectl describe clusterrolebinding system:editor 

Name:         system:editor
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"rbac.authorization.k8s.io/v1beta1","kind":"ClusterRoleBinding","metadata":{"annotations":{},"creationTimestamp":null,"name"...
Role:
  Kind:  ClusterRole
  Name:  edit
Subjects:
  Kind   Name           Namespace
  ----   ----           ---------
  Group  system:editor  
```


Do the same for bindig ClusterRole `view` with ClusterRoleBinding `system:viewer`
```
kubectl create clusterrolebinding system:viewer \
    --clusterrole=view \
    --group=system:viewer \
    --dry-run -o yaml > clusterrolebinding_system_viewer.yaml

cat clusterrolebinding_system_viewer.yaml
```

Apply
```
kubectl apply -f clusterrolebinding_system_viewer.yaml
```

# 7.6 Restrict AWS IAM User Access by Binding them to Right K8s User Groups (which is bound to right ClusterRole)

Swtich back to the original AWS PROFILE first.
```sh
export AWS_PROFILE=YOUR_ORIGINAL_PROFILE
```

Then edit configmap
```
kubectl edit -n kube-system configmap/aws-auth
```

This time, instead of specifying `system:masters`, use `system:viewer` K8s user group created from new ClusterRoleBinding `system:viewer`
```yaml
mapUsers: |
    - userarn: arn:aws:iam::111122223333:user/eks-viewer  # AWS IAM User
      username: eks-viewer
      groups:
      - system:viewer  # K8s User Group which is tied to ClusterRole
```

Check if AWS IAM user `eks-viewer` can now view K8s cluster
```sh
# switch AWS IAM user by changing AWS profile
export AWS_PROFILE=eks-viewer

# do some READ/GET operations with kubectl
kubectl get pod
NAME                 READY   STATUS    RESTARTS   AGE
guestbook-dxkpd      1/1     Running   0          4h6m
guestbook-fsqx8      1/1     Running   0          4h6m
guestbook-nnrjc      1/1     Running   0          4h6m
redis-master-6dbj4   1/1     Running   0          4h8m
redis-slave-c6wtv    1/1     Running   0          4h7m
redis-slave-qccp6    1/1     Running   0          4h7m
```

Check authorization of this AWS IAM user `eks-viewer`, bound to K8s user group `system:viewer`
```
$ kubectl auth can-i create deployments
no

$ kubectl auth can-i delete deployments
no

$ kubectl auth can-i delete ClusterRole
no
```

If you try to create, you will get an expected permission error because `eks-viewer` AWS IAM user is bound to `system:viewer` K8s user group, which has only `view` ClusterRole permissions
```
# try to create namespace
$ kubectl create namespace test

Error from server (Forbidden): namespaces is forbidden: User "eks-viewer" cannot create resource "namespaces" in API group "" at the cluster scope
```


This time, instead of specifying `system:masters`, use `system:viewer` K8s user group created from new ClusterRoleBinding `system:viewer`
```yaml
mapUsers: |
    - userarn: arn:aws:iam::xxxxxxxxx:user/eks-viewer # AWS IAM User
      username: eks-viewer
      groups:
      - system:viewer  # K8s User Group
    - userarn: arn:aws:iam::xxxxxxxxxxx:user/eks-editor # AWS IAM User
      username: eks-editor
      groups:
      - system:editor  # K8s User Group
```

Check if AWS IAM user `eks-editor` can now view K8s cluster
```sh
# switch AWS IAM user by changing AWS profile
export AWS_PROFILE=eks-editor

# do some READ/GET operations with kubectl
kubectl get pod
NAME                 READY   STATUS    RESTARTS   AGE
guestbook-dxkpd      1/1     Running   0          4h6m
guestbook-fsqx8      1/1     Running   0          4h6m
guestbook-nnrjc      1/1     Running   0          4h6m
redis-master-6dbj4   1/1     Running   0          4h8m
redis-slave-c6wtv    1/1     Running   0          4h7m
redis-slave-qccp6    1/1     Running   0          4h7m
```

Check authorization of this AWS IAM user `eks-editor`, bound to K8s user group `system:editor`
```
$ kubectl auth can-i create deployments
yes

$ kubectl auth can-i delete deployments
yes

$ kubectl auth can-i delete ClusterRole
no

$ kubectl auth can-i create namespace
no
```
## What Just Happened?
# 7.7 Allow AWS IAM Role to K8s Cluster
In reality, AWS IAM user should be assuming roles because IAM role's temporary credentials get rotated every hour or so  and thus safer.

To allow AWS IAM role to K8s, simply add IAM Role ARNs to `aws-auth` configmap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::xxxxxxxx:role/eksctl-eks-from-eksctl-nodegroup-NodeInstanceRole-R3EFEQC9U6U
      username: system:node:{{EC2PrivateDNSName}}
    - groups:  # <----- new IAM role entry
      - system:viewer  # <----- K8s User Group
      rolearn: arn:aws:iam::xxxxxxxx:role/eks-viewer-role  # AWS IAM role
      username: eks-viewer
    - groups:  # <----- new IAM role entry
      - system:editor   # <----- K8s User Group
      rolearn: arn:aws:iam::xxxxxxxx:role/eks-editor-role  # AWS IAM role
      username: eks-editor
 ```
   # it's best practice to use IAM role rather than IAM user's access key
  # mapUsers: |
  #   - userarn: arn:aws:iam::xxxxxxxxx:user/eks-viewer
  #     username: eks-viewer
  #     groups:
  #     - system:viewer
  #   - userarn: arn:aws:iam::xxxxxxxx:user/eks-editor
  #     username: eks-editor
  #     groups:
  #     - system:editor

# Pod level authenication and authorization
<img width="340" alt="image" src="https://user-images.githubusercontent.com/75510135/144586910-cb9f72c2-c515-4cce-a2bf-3f841d487875.png">

# 1. Recap of Pod Authentication within K8s cluster using Service Account


1. pod is associated with a service account, and a credential (token) for that service account is placed into the filesystem each container in that pod, at `/var/run/secrets/kubernetes.io/serviceaccount/token`
2. Pods can authenticate with API server using an auto-mounted token in the default service account and secret that only the Kubernetes API server could validate. 

<img width="525" alt="image" src="https://user-images.githubusercontent.com/75510135/144588537-addd9a52-d69d-4fd8-bf4e-74c63b63a7d4.png">

# 2. Recap of Pod Authorization to AWS Resources using EKS Node's Instance Profile



1. Pod runs on EKS worker nodes, which have AWS IAM instance profile attached to them.
2. kubelet agent in worker nodes get temporary IAM credentials from IAM role attached to worker nodes through instance metadata at `169.254.169.254/latest/meta-data/iam/security-credentials/IAM_ROLE_NAME`

```
kubectl run --image curlimages/curl --restart Never --command curl -- /bin/sh -c "sleep 500"
```
# 3. IRSA Architecture Overview
ref: https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/

1. When you launch a pod with `kubectl apply -f`, the YAML manifest is submitted to the API server with the Amazon EKS Pod Identity webhook configured
2. Because the service account `irsa-service-account` has an eks.amazonaws.com/role-arn annotation, the webhook __injects the necessary environment variables (AWS_ROLE_ARN and AWS_WEB_IDENTITY_TOKEN_FILE)__ and sets up the __aws-iam-token projected volume__ (this is not IAM credentials, just JWT token) in the pod that the job supervises.
3. Service accoount associated with the pod authenticate to OIDC (which in turns authenticate to AWS IAM service on behalf of service account) and get JWT token back from OIDC and saves it in `AWS_WEB_IDENTITY_TOKEN_FILE`
4. When container executes `aws s3 ls`, the pod performs an `sts:assume-role-with-web-identity` with the token stored in `AWS_WEB_IDENTITY_TOKEN_FILE` path to __assume the IAM role__ (behind the scene,). It receives temporary credentials that it uses to complete the S3 write operation.



# setup 1: Create IAM assumable role which specifies namespace and service account, and OIDC endpoint

With `eksctl create cluster`, OIDC proviser URL is already created so you can skip this:
```bash
# get the cluster’s identity issuer URL
ISSUER_URL=$(aws eks describe-cluster \
                       --name eks-from-eksctl \
                       --query cluster.identity.oidc.issuer \
                       --output text)

# create OIDC provider
aws iam create-open-id-connect-provider \
          --url $ISSUER_URL \
          --thumbprint-list $ROOT_CA_FINGERPRINT \
          --client-id-list sts.amazonaws.com
```


Associate OIDC provider with cluster
```bash
eksctl utils associate-iam-oidc-provider \
            --region=us-west-2 \
            --cluster=eks-from-eksctl \
            --approve
            ```

# setup 2: Annotate k8s service account with IAM role ARN

1. creates an IAM role and attaches the specified policy to it, in our case `arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess`.
2. creates a Kubernetes service account, `irsa-service-account` here, and annotates the service account with the IAM role.

```bash
eksctl create iamserviceaccount \
                --name irsa-service-account \
                --namespace default \
                --cluster eks-from-eksctl \
                --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
                --approve \
                --region us-west-2
  ```

You can see a new IAM role `eksctl-eks-from-eksctl-addon-iamserviceaccou-Role1-1S8X0CMRPPPLY` created in Console.

You can also see details of service account `irsa-service-account`
```bash
$ kubectl describe serviceaccount irsa-service-account

```

# setup 3: Reference service account from Pod yaml

Create a deployment yaml which uses `aws/aws-cli` docker image and `irsa-service-account` service account
```bash
kubectl run irsa-iam-test \
    --image amazon/aws-cli  \
    --serviceaccount irsa-service-account \
    --dry-run -o yaml \
    --command -- /bin/sh -c "sleep 500" \
    > deployment_irsa_test.yaml
```
Create deployment
```
kubectl apply -f deployment_irsa_test.yaml
```

Shell into a container to verify this pod has proper S3 read permissions by assuming the IAM role
```bash
kubectl exec -it irsa-iam-test-cf8d66797-kx2s2  sh

# show env
sh-4.2# env

AWS_ROLE_ARN=arn:aws:iam::xxxxxx:role/eksctl-eks-from-eksctl-addon-iamserviceaccou-Role1-1S8X0CMRPPPLY  # <--- the created IAM role ARN is injected
GUESTBOOK_PORT_3000_TCP_ADDR=10.100.53.19
HOSTNAME=irsa-iam-test-cf8d66797-kx2s2
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token # <---- this is the JWT token to authenticate to OIDC, and then OIDC will assume IAM role using AWS STS

# get aws version of docker container
sh-4.2# aws --version
aws-cli/2.0.22 Python/3.7.3 Linux/4.14.181-140.257.amzn2.x86_64 botocore/2.0.0dev26

# list S3
sh-4.2# aws s3 ls
2020-06-13 16:27:43 eks-from-eksctl-elb-access-log
```

Check IAM identity of the IAM role assumed by pod
```
sh-4.2# aws sts get-caller-identity
{
    "UserId": "AROAS6KA4SFRWOLUNLZAK:botocore-session-1592141699",
    "Account": "xxxxxx",
    "Arn": "arn:aws:sts::xxxxxx:assumed-role/eksctl-eks-from-eksctl-addon-iamserviceaccou-Role1-1S8X0CMRPPPLY/botocore-session-1592141699"
}
```

## What Just Happened?!


1. When you launch a pod with `kubectl apply -f`, the YAML manifest is submitted to the API server with the Amazon EKS Pod Identity webhook configured
2. Because the service account `irsa-service-account` has an eks.amazonaws.com/role-arn annotation, the webhook __injects the necessary environment variables (AWS_ROLE_ARN and AWS_WEB_IDENTITY_TOKEN_FILE)__ and sets up the __aws-iam-token projected volume__ (this is not IAM credentials, just JWT token) in the pod that the job supervises.
3. Service accoount associated with the pod authenticate to OIDC (which in turns authenticate to AWS IAM service on behalf of service account) and get JWT token back from OIDC and saves it in `AWS_WEB_IDENTITY_TOKEN_FILE`
4. When container executes `aws s3 ls`, the pod performs an `sts:assume-role-with-web-identity` with the token stored in `AWS_WEB_IDENTITY_TOKEN_FILE` path to __assume the IAM role__ (behind the scene,). It receives temporary credentials that it uses to complete the S3 write operation.


## But Don't Stop Here! Pod can still access node's instance profile (which has too much permissions)

Try to access instance metadata endpoint
```sh
# get a shell into a container in pod
kubectl exec -it irsa-iam-test-cf8d66797-hc5f9  sh

curl 169.254.169.254/latest/meta-data/iam/security-credentials/eksctl-eks-from-eksctl-nodegroup-NodeInstanceRole-R3EFEQC9U6U

```


# 4. Block Access to Instance Metadata

SSH into EKS worker nodes
```sh
eval $(ssh-agent)
ssh-add -k ~/.ssh/
ssh -A ec2-user@192.168.20.213
```

Run commands to change iptables
```bash
yum install -y iptables-services
iptables --insert FORWARD 1 --in-interface eni+ --destination 169.254.169.254/32 --jump DROP
iptables-save | tee /etc/sysconfig/iptables 
systemctl enable --now iptables
```

# 5. Limitations with eksctl and eksctl Manated Nodes
Ideally, you should add the above script to userdata script in launch configuration so all the worker nodes starting up will execute shell commands.

Also, there are __some downsides__ to managed node groups ([eksctl doc](https://eksctl.io/usage/eks-managed-nodes/#feature-parity-with-unmanaged-nodegroups)):
> Control over the node bootstrapping process and customization of the kubelet are not supported. This includes the following fields: classicLoadBalancerNames, maxPodsPerNode, __taints__, targetGroupARNs, preBootstrapCommands, __overrideBootstrapCommand__, clusterDNS and __kubeletExtraConfig__.


