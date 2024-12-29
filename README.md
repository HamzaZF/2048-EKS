
# Game 2048 on AWS EKS with Fargate

This repository demonstrates how to deploy a scalable, containerized version of the popular 2048 game on AWS Elastic Kubernetes Service (EKS) with Fargate support. It utilizes Kubernetes manifests for deployment and leverages the AWS Load Balancer Controller for external access.

## Features

- **EKS with Fargate:** Serverless Kubernetes using AWS Fargate profiles.
- **Load Balancer Controller:** Efficient traffic routing with an Application Load Balancer (ALB).
- **High Availability:** Configured with 5 replicas for redundancy and performance.
- **Resource Optimization:** Fine-tuned memory and CPU resource requests and limits.

---

## Prerequisites

1. AWS CLI configured with appropriate permissions.
2. `eksctl` installed.
3. `kubectl` installed and configured.
4. `Helm` installed for Kubernetes package management.

---

## Deployment Guide

### 1. Create an EKS Cluster with Fargate
```bash
eksctl create cluster --name <cluster-name> --region <region> --fargate
```

### 2. Update kubeconfig
```bash
aws eks update-kubeconfig --name <cluster-name> --region <region>
```

### 3. Associate IAM OIDC Provider
```bash
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve
```

### 4. Create IAM Policy for AWS Load Balancer Controller
Download the IAM policy document from the official AWS Load Balancer Controller repository:

```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/docs/install/iam_policy.json
```

Then, create the IAM policy:

```bash
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam-policy.json
```

### 5. Create IAM Service Account
```bash
eksctl create iamserviceaccount \
  --cluster=<cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name <role-name> \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### 6. Install AWS Load Balancer Controller
```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<vpc-id>
```

### 7. Create Fargate Profile
```bash
eksctl create fargateprofile --cluster <cluster-name> --region <region> --name <fargate-profile-name> --namespace <namespace>
```

### 8. Deploy Kubernetes Resources
```bash
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

---

## Kubernetes Manifests

### Namespace (`namespace.yaml`)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
```

### Deployment (`deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-2048
  namespace: game-2048
spec:
  replicas: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
        - image: public.ecr.aws/l6m2t8p7/docker-2048:latest
          imagePullPolicy: Always
          name: app-2048
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

### Service (`service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: app-2048
```

### Ingress (`ingress.yaml`)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-2048
                port:
                  number: 80
```

---

## Clean Up
To delete all resources and avoid incurring further charges:
```bash
eksctl delete cluster --name <cluster-name>
```

---

## Authors

- [Your Name](https://github.com/your-github-username)

## License

This project is licensed under the MIT License.
