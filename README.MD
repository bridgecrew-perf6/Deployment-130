# Deployment Of NodeJs on EKS using Fargate

## Step 1: Creating Ec2 instance
- Create an EC2 instance using aws console (ubuntu t2.micro) and download the private key
- Name the private key file ec2-control.pem.
- Move that file to home folder.
    ```bash
    mv <path to the private key file> ~
    ```
- Restrict all of the permissions of that file except read.
    ```bash
    chmod 400 ~/ec2-control.pem
    ```
- SSH into ec2 instance using that private key file.
    ```bash
    ssh -i ~/ec2-control.pem ubuntu@<ip>
    ```
    <b>Note</b>: ip is the IPv4 address of Ec2 instance. This can be copied from the aws console.
- 
## Step 2: Installing Dependencies in the the VM.
- In the EC2 install <a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html">aws-cli</a>, <a href="https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html">eksctl</a>, <a href="https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/">kubectl</a> and <a href="https://helm.sh/docs/intro/install/#from-script">helm</a>.
- Log into aws-cli using IAM user (using IAM user with restricted permissions is recommended)
    ```bash
    aws configure
    ```
## Step 3: Creating cluster and configuring kubectl
- Create cluster using eksctl cli with all the configurations and fargate profiles in the [cluster-config.yml](cluster-config.yml) file.
    ```bash
    eksctl create cluster -f cluster-config.yml
    ```
    ### For more info: https://eksctl.io/usage/fargate-support/
- Configure kubectl to access the created cluster:
    ```bash
    aws eks update-kubeconfig --region region-code --name cluster-name
    ```
- Check the configuration using:
    ```bash
    kubectl get svc
    ```
    Output
    ```bash
    NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    svc/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   1m
    ```
    ### For more info: https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html
## Step 4: Creating IAM Role and Service Account for Load Balancer
- Create IAM OIDC provider:
    ```bash
    eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster ${CLUSTER_NAME} \
    --approve
    ```
- Download IAM policy for load-balancer (excluding us-east)
    ```bash
    curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.1/docs/install/iam_policy.json
    ```

- Create a policy called AWSLoadBalancerControllerIAMPolicy
    ```bash
    aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
    ```
- Create IAM service account
    ```bash
    eksctl create iamserviceaccount \
  --cluster=${CLUSTER_NAME} \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name "AmazonEKSLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
  ```
- Annotate your service account
    ```bash
    kubectl annotate serviceaccount -n kube-system aws-load-balancer-controller \
    eks.amazonaws.com/sts-regional-endpoints=true
    ```
## Step 5: Installing Load-Balancer using helm
- Add the eks-charts repository.
    ```bash
    helm repo add eks https://aws.github.io/eks-charts
    ```
- Update local repo
    ```bash
    helm repo update
    ```
- Install the AWS Load Balancer Controller
    ```bash
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=cluster-name \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller  \
  --set region=region-code
  --set vpcId=vpc-xxxxxxxx
  ```
- Verify that controller is installed successfully
    ```bash
    kubectl get deployment -n kube-system aws-load-balancer-controller
    ```
    Output
    ```bash
    NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
    aws-load-balancer-controller   2/2     2            2           34s
    ```

## Step 6: Create Config-Map yaml file
- Create config-map.yml file
    ```bash
    nano config-map.yml
    ```
- Copy the contents of [config-map.yml](config-map.yml), write your environment variables under data section save the file (Ctrl+X).
- Apply the file using kubectl.
    ```bash
    kubectl apply -f config-map.yml
    ```
## Step 7: Create Service and Deployment
- Create a file named nginx.yml and open it in text editor.
    ```bash
    nano nginx.yml
    ```
- Copy the contents of [nginx.yml](nginx.yml) and make necessary changes according to your application.
- Apply this file using kubectl
    ```bash
    kubectl apply -f nginx.yml
    ```
- Check the service and deployments using kubectl.
    ```bash
    kubectl get <service/deployment> --all-namespaces
    ```
    For a particular namespace
    ```bash
    kubectl get <service/deployment> -namespace <namespace>
    ```
## Step 8: Create Ingress Controller
- Create ingress.yml and copy the contents of [ingress.yml](ingress.yml) in it.
    ```bash
    nano ingress.yml
    ```
- Apply this ingress file using kubectl.
    ```bash
    kubectl apply -f ingress.yml
    ```
- Check the ingress controller is running and get the url for the application.
    ```bash
    kubectl get ingress --all-namespaces
    ```
    Output
    ```bash
    NAMESPACE   NAME              CLASS    HOSTS   ADDRESS                                                                 PORTS   AGE
    default     ingress-backend   <none>   *       k8s-default-ingressb-2d7bfa3e15-213467671.eu-west-3.elb.amazonaws.com   80      2d10h
```
