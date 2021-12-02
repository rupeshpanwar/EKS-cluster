- You can watch the status by running 'kubectl --namespace default get services -o wide -w nginx-ingress-controller-ingress-nginx-controller'

<img width="721" alt="image" src="https://user-images.githubusercontent.com/75510135/144007424-f112be5d-2920-4637-891e-d3dab5dd61c3.png">

<img width="697" alt="image" src="https://user-images.githubusercontent.com/75510135/144018368-036c6f13-91e7-439d-a6a7-9f4cadffc5f6.png">

<img width="690" alt="image" src="https://user-images.githubusercontent.com/75510135/144033786-295c7e4f-bb9e-43cf-a669-ad32c11bfedb.png">

<img width="673" alt="image" src="https://user-images.githubusercontent.com/75510135/144209095-d202e5e3-f3c0-4e5a-bec8-ea3c9181db96.png">
<img width="663" alt="image" src="https://user-images.githubusercontent.com/75510135/144214075-988f108c-1dfe-45fb-9ecc-50bf3ed5d2c0.png">

Get `guestbook` service 
```
kubectl get service guestbook
```
There are a few __downsides__ of service of type `LoadBalancer`:
- is L4 load balancer, therefore has no knowledge about L7 (i.e. http path and host).
- is costly as each new service creates a new load balancer (one to one mapping between service and load balancer)


# 6.1 Create Ingress Controller (Nginx) using Helm Chart
Refs: 
- https://github.com/helm/charts/tree/master/stable/nginx-ingress
- https://kubernetes.github.io/ingress-nginx/deploy/
- https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx


```sh
kubectl create namespace nginx-ingress-controller

# stable/nginx-ingress is deprecated 
# helm install nginx-ingress-controller stable/nginx-ingress -n nginx-ingress-controller

# add new repo ingress-nginx/ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add stable https://charts.helm.sh/stable
helm repo update

# install
helm install nginx-ingress-controller ingress-nginx/ingress-nginx
```
- You can watch the status by running 'kubectl --namespace default get services -o wide -w nginx-ingress-controller-ingress-nginx-controller'

Check pods, services, and deployments created in `nginx-ingress-controller` namespace
```
kubectl get pod,svc,deploy -n nginx-ingress-controller
```
You see `nginx-ingress-controller-controller` service of type `LoadBalancer`. This is still L4 load balancer but `nginx-ingress-controller-controller` pod is running nginx inside to do L7 load balancing inside EKS cluster.

# 6.2 Create Ingress resource for L7 load balancing by http hosts & paths

Create ingress resource
```bash
kubectl apply -f ingress.yaml
```

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    # nginx.ingress.kubernetes.io/backend-protocol: "HTTPS" # this is the case where a container accept HTTPS
  name: guestbook
  namespace: default
spec:
  #  this is in case SSL termination happens at nginx ingress controller
  # tls:
  # - hosts:
  #   - a92226b56ed6548beaae3bcc7e3b9a5c-997944382.us-west-2.elb.amazonaws.com
  #   secretName: tls-secret 
  rules:
    - http:
        paths:
          - backend:
              serviceName: guestbook
              servicePort: 3000 
            path: /
   ```
  
Get the public DNS of AWS ELB created from the `nginx-ingress-controller-controller` service
```bash
kubectl  get svc nginx-ingress-controller-controller -n nginx-ingress-controller | awk '{ print $4 }' | tail -1
```


Now modify `guestbook` service type from `LoadBalancer` to `NodePort`.

First get yaml 
```bash
kubectl get svc guestbook -o yaml
```


Delete the existing `guestbook` service as service is immutable
```bash
kubectl delete svc guestbook
```

Then apply new service
```bash
kubectl apply -f service_guestbook_nodeport.yaml
```
Check services in `default` namespace
```bash
$ kubectl get svc

NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
guestbook      NodePort    10.100.53.19    <none>        3000:30605/TCP   20s
kubernetes     ClusterIP   10.100.0.1      <none>        443/TCP          3h38m
redis-master   ClusterIP   10.100.174.46   <none>        6379/TCP         77m
redis-slave    ClusterIP   10.100.103.40   <none>        6379/TCP         76m
```

Lastly, check ingress controller's public DNS is reachable from browser
```bash
# visit the URL from browser
kubectl  get svc nginx-ingress-controller-controller -n nginx-ingress-controller | awk '{ print $4 }' | tail -1
```

# 6.3  What Just Happened?

1. Replaced `guestbook` service of type `LoadBalancer` to of `NodePort`
2. Front `guestbook` service with `nginx-ingress-controller` service of type `LoadBalancer`
3. `nginx-ingress-controller` pod will do L7 load balancing based on HTTP path and host
4. Now you can create multiple services and bind them to one ingress controller (one AWS ELB)


# 6.4 (BEST PRACTICE) Enable HTTPS for AWS ELB (SSL Termination at ELB)
If you visit the public DNS of ELB with https, you will see a fake Nginx certificate

This is the default cert configured in `/etc/nginx/nginx.conf` inside `nginx-ingress-controller-controller` pod
```sh
# ssh into the pod
kubectl exec -it nginx-ingress-controller-controller-767d5fd45d-q7cpw -n nginx-ingress-controller sh

# grep for "ssl"
$ grep -r "ssl" *.conf

nginx.conf:                     is_ssl_passthrough_enabled = false,
nginx.conf:             listen_ports = { ssl_proxy = "442", https = "443" },
nginx.conf:     ssl_certificate     /etc/ingress-controller/ssl/default-fake-certificate.pem; # <-- here fake cert
nginx.conf:     ssl_certificate_key /etc/ingress-controller/ssl/default-fake-certificate.pem;
```

This is the case 2 [SSL terminates at Nginx Ingress Controller pod]

To achieve SSL termination at AWS ELB (case 1 in the diagram), you need to provision SSL cert and associate it with ELB.


## 1. Create Self-signed Server Certificate (1024 or 2048 bit long)

Refs: 
- https://kubernetes.github.io/ingress-nginx/user-guide/tls/#tls-secrets
- https://kubernetes.github.io/ingress-nginx/user-guide/tls/#tls-secretshttps://stackoverflow.com/questions/10175812/how-to-create-a-self-signed-certificate-with-openssl

Since we don't own any domains for this course, we need to create a self-signed cert by ourselves.
```
openssl req \
        -x509 \
        -newkey rsa:2048 \
        -keyout elb.amazonaws.com.key.pem \
        -out elb.amazonaws.com.cert.pem \
        -days 365 \
        -nodes \
        -subj '/CN=*.elb.amazonaws.com'
 ```       

Check contents of cert
```sh
openssl x509 -in elb.amazonaws.com.cert.pem -text -noout
```
## 2. Import Server Certificate to ACM (Amazon Certificate Manager)
Ref: https://docs.aws.amazon.com/acm/latest/userguide/import-certificate-api-cli.html

```bash
aws acm import-certificate \
  --certificate fileb://elb.amazonaws.com.cert.pem \
  --private-key fileb://elb.amazonaws.com.key.pem \
  --region ap-southeast-1
```

You can see the new cert in AWS ACM console.


## 3. Add AWS ACM ARN to Nginx ingress controller's service annotations in overrides.yaml
Refs:
- https://kubernetes.io/docs/concepts/services-networking/service/#ssl-support-on-aws
- https://github.com/helm/charts/tree/master/stable/nginx-ingress

Edit and replace `YOUR_CERT_ARN` with your cert ARN in [nginx_helm_chart_overrides_ssl_termination_at_elb.yaml](nginx_helm_chart_overrides_ssl_termination_at_elb.yaml)

```yaml
controller:
  service:
    annotations:
      # https for AWS ELB. Ref: https://kubernetes.io/docs/concepts/services-networking/service/#ssl-support-on-aws
      service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "YOUR_CERT_ARN"
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp" # backend pod doesn't speak HTTPS
      service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443" # unless ports specified, even port 80 will be SSL provided
    targetPorts:
      https: http # TSL Terminates at the ELB
  config:
    ssl-redirect: "false" # don't let https to AWS ELB -> http to Nginx redirect to -> https to Nginx
```

## 4. Upgrade Nginx ingress controller Helm chart
```bash
helm upgrade nginx-ingress-controller \
    stable/nginx-ingress \
    -n nginx-ingress-controller \
    -f nginx_helm_chart_overrides_ssl_termination_at_elb.yaml
```

Check `nginx-ingress-controller-controller` service in `nginx-ingress-controller` namespace have annotation added
```sh
kubectl describe svc nginx-ingress-controller-controller -n nginx-ingress-controller
```
Visit the public DNS of ELB again from browser
```bash
# visit the URL from browser
kubectl  get svc nginx-ingress-controller-controller -n nginx-ingress-controller | awk '{ print $4 }' | tail -1

```
## 5. How to Fix "400 Bad Request. The plain HTTP request was sent to HTTPS port"

Same if you curl 
```bash
curl https://a04081d9fc1494d16a1c94d1d4a3759a-430198622.ap-southeast-1.elb.amazonaws.com/ -v -k
```
This is because by default, Nginx Ingress Controller's _service_ definition has `controller.service.targetPorts.https=443` or in yaml format (https://github.com/helm/charts/tree/master/stable/nginx-ingress),

```sh
service:
    targetPorts:
      http: 80
      https: 443  # <--- this means incoming HTTPs to ELB will be sent to backend (Nginx Ingress Controller) 443
```
# 6.5 (BEST PRACTICE) Enable ELB access logs by K8s service annotations
Creating AWS ELB from K8s ingress controller isn't enough for production.

You need to enable access logs of ELB.


1. Create S3 bucket for ELB to send access logs to
```bash
# from S3 console, create this bucket
aws s3 mb s3://eks-from-eksctl-elb-access-log-0212
```
2. Create S3 bucket policy so that ELB can push it to the bucket
Ref: https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/enable-access-logs.html

Given this template [s3_bucket_policy_elb_access_log.json](s3_bucket_policy_elb_access_log.json),
you need to interpolate `elb-account-id` based on your region, `your-aws-account-id`, `bucket-name`, and `prefix`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::elb-account-id:root"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::bucket-name/prefix/AWSLogs/your-aws-account-id/*"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::bucket-name/prefix/AWSLogs/your-aws-account-id/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::bucket-name"
    }
  ]
}
```


3. Add k8s service annotations to `nginx-ingress-controller` service so it knows access log bucket
Ref: https://kubernetes.io/docs/concepts/services-networking/service/#elb-access-logs-on-aws

These annotations need to be added to `nginx-ingress-controller` service
```yaml
metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
        # Specifies whether access logs are enabled for the load balancer
        service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval: "60"
        # The interval for publishing the access logs. You can specify an interval of either 5 or 60 (minutes).
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "my-bucket"
        # The name of the Amazon S3 bucket where the access logs are stored
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "my-bucket-prefix/prod"
        # The logical hierarchy you created for your Amazon S3 bucket, for example `my-bucket-prefix/prod`
```


To add this, add above to `nginx_helm_chart_overrides_access_logs.yaml` which will be passed when installing `nginx` helm chart.

[nginx_helm_chart_overrides_access_logs.yaml](nginx_helm_chart_overrides_access_logs.yaml)

4. Upgrade Nginx ingress controller specifying overriding yaml file with `-f` option (ideally in production, you would add service annotations before first install)

```sh
# upgrade it with -f nginx_helm_chart_overrides_access_logs.yaml
helm upgrade nginx-ingress-controller \
    stable/nginx-ingress \
    -f nginx_helm_chart_overrides_access_logs.yaml \
    -n nginx-ingress-controller 

```
<img width="894" alt="image" src="https://user-images.githubusercontent.com/75510135/144390812-ee663caf-28d9-477e-b507-e7a4788d779b.png">

<img width="656" alt="image" src="https://user-images.githubusercontent.com/75510135/144390865-1b043932-bbf2-46a4-baf3-21a5aecf5ef7.png">


