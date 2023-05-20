# Prerequisites

* [Amazon VPC CNI plugin](https://github.com/aws/amazon-vpc-cni-k8s)
* [OIDC](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
* [eksctl](https://eksctl.io)
* [helm](https://helm.sh)

Personally, I prefer to create NLBs using IP mode. In the mode, the NLB delivers traffic directly to pods behind the service, eliminating the need for an extra network hop through the worker nodes in the Kubernetes cluster.

### AWS Load Balancer Controller Installation
* Create an IAM policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

* Install AWS Load Balancer Controller using helm
  - Find the VPC ID
    ```
    aws ec2 describe-vpcs \
        --filters "Name=tag:Name,Values=management*" \
        --region us-east-1 \
        --query "Vpcs[*].VpcId" \
        --output text

    # Output
    vpc-066f8d461f2a18600
    ```

  - Install AWS Load Balancer Controller

    Replace ${AWS_ACCOUNT_ID} in aws-load-balancer-controller/irsa.yaml with your own.
    ```
    # IRSA
    eksctl create iamserviceaccount \
        --override-existing-serviceaccounts \
        -f aws-load-balancer-controller/irsa.yaml \
        --approve

    # To install a chart
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update
    helm upgrade -i -n kube-system --version 1.5.2 \
        -f aws-load-balancer-controller/values.yaml \
        aws-load-balancer-controller eks/aws-load-balancer-controller
    ```

  - Verify that the controller is installed
    ```
    kubectl get deployment -n kube-system aws-load-balancer-controller

    # The output is as follows
    NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
    aws-load-balancer-controller   1/1     1            1           35s
    ```

* Create an **External** NLB

    The _external_ NLB will be provisioned in IP mode, and preserve the source IP address of clients.

    Here is the manifest snippet:
    ```
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: preserve_client_ip.enabled=true
    ```

    Apply the manifest `external-ip-mode.yaml` to the cluster:
    ```
    kubectl apply -f sample-manifests/external-ip-mode.yaml

    # Verify that the service was deployed
    kubectl get svc nginx

    # The output is as follows
    NAME    TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
    nginx   LoadBalancer   172.20.124.119   k8s-default-nginx-8fb58cacd9-4356647eb000f167.elb.us-east-1.amazonaws.com   80:30466/TCP   43s
    ```

    Send traffic to the service:
    ```
    curl k8s-default-nginx-8fb58cacd9-4356647eb000f167.elb.us-east-1.amazonaws.com

    # Check the client's IP
    kubectl logs -f -l app=nginx

    # The output is as follows
    2023/05/19 11:08:40 [notice] 1#1: start worker processes
    2023/05/19 11:08:40 [notice] 1#1: start worker process 29
    2023/05/19 11:08:40 [notice] 1#1: start worker process 30
    61.56.135.200 - - [19/May/2023:11:11:57 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.84.0" "-"
    ```

* Create an **Internal** NLB

    The _internal_ NLB will be provisioned in IP mode.

    Here is the manifest snippet:
    ```
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb-ip"
    ```

    Apply the manifest `internal-ip-mode.yaml` to the cluster:
    ```
    kubectl apply -f sample-manifests/internal-ip-mode.yaml

    # Verify that the service was deployed
    kubectl get svc nginx-int

    # The output is as follows
    NAME        TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE
    nginx-int   LoadBalancer   172.20.76.110   k8s-default-nginxint-3fa97b9ea2-6288b683176e61d5.elb.us-east-1.amazonaws.com   80:30146/TCP   2m31s
    ```

### Ingress Controller Installation
I recommend deploying multiple Ingress controllers with separate helm charts.

**Note:** ACM certificates must be requested or imported in the same AWS Region as your Load Balancer.

* **External** Ingress Controller

Replace ${AWS_ACCOUNT_ID} in ingress-nginx/values.yaml with your own.
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade -i --create-namespace \
    -n ingress-nginx \
    --version 4.6.1 \
    -f ingress-nginx/values.yaml \
    ingress-nginx ingress-nginx/ingress-nginx

# Verify that the service was deployed
kubectl get svc -n ingress-nginx ingress-nginx-controller

# The output is as follows
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   172.20.167.196   k8s-ingressn-ingressn-2e2fc803ab-5cd4d026da82e606.elb.us-east-1.amazonaws.com   80:32732/TCP,443:32075/TCP   78s
ingress-nginx-controller-admission   ClusterIP      172.20.119.164   <none>                                                                          443/TCP                      78s
ingress-nginx-defaultbackend         ClusterIP      172.20.249.114   <none>                                                                          80/TCP                       78s
```

* **Internal** Ingress Controller
```
helm upgrade -i --create-namespace \
    -n ingress-nginx \
    --version 4.6.1 \
    -f ingress-nginx/internal-values.yaml \
    ingress-nginx-int ingress-nginx/ingress-nginx

# Verify that the service was deployed
kubectl get svc -n ingress-nginx ingress-nginx-int-controller-internal

# The output is as follows
NAME                                    TYPE           CLUSTER-IP      EXTERNAL-IP                                                                     PORT(S)                      AGE
ingress-nginx-int-controller-internal   LoadBalancer   172.20.239.61   k8s-ingressn-ingressn-b2ef81929b-a362101fffb8a8df.elb.us-east-1.amazonaws.com   80:31431/TCP,443:32381/TCP   56s
```
