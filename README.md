# FreshMart Supermarket Demo — Docker + AWS EKS

This package containerizes the existing static FreshMart website with NGINX and includes Kubernetes manifests and a Jenkins pipeline example.

## What's included
- `index.html` — the existing supermarket frontend demo
- `Dockerfile` — packages the site into an NGINX container
- `k8s/namespace.yaml` — app namespace
- `k8s/deployment.yaml` — two replicas, rolling update, probes, resource limits and restricted container security context
- `k8s/service.yaml` — public AWS LoadBalancer Service (creates AWS resources and may incur charges)
- `Jenkinsfile` — example pipeline to build, push to ECR, and roll out to EKS

## 1. Test locally with Docker
```bash
docker build -t freshmart-web:local .
docker run --rm -p 8080:80 freshmart-web:local
```
Open `http://localhost:8080`.

## 2. Create an ECR repository
Set your AWS region and account ID first:
```bash
export AWS_REGION=ap-south-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REPOSITORY=freshmart-web
aws ecr create-repository --repository-name "$ECR_REPOSITORY" --image-scanning-configuration scanOnPush=true --region "$AWS_REGION"
```
If it already exists, the create command may return an already-exists error; continue with the login/build steps.

## 3. Build and push an image
```bash
export IMAGE_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:v1"
aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
docker build --pull -t "$IMAGE_URI" .
docker push "$IMAGE_URI"
```

## 4. Configure Kubernetes access
Your EKS cluster must already exist, have healthy worker nodes, and your AWS identity must be authorized in the cluster:
```bash
aws eks update-kubeconfig --name YOUR_EKS_CLUSTER_NAME --region "$AWS_REGION"
kubectl get nodes
```

## 5. Deploy
Replace `REPLACE_WITH_ECR_URI:latest` in `k8s/deployment.yaml` with your full ECR image URI, for example `123456789012.dkr.ecr.ap-south-1.amazonaws.com/freshmart-web:v1`. Then run:
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl rollout status deployment/freshmart-web -n freshmart
kubectl get pods,svc -n freshmart
```
Wait for the LoadBalancer Service's `EXTERNAL-IP`/hostname:
```bash
kubectl get svc freshmart-web -n freshmart --watch
```
Open `http://<load-balancer-hostname>` once provisioned.

## 6. Jenkins setup
- Put these files in a Git repository and configure a Jenkins Pipeline job to use its `Jenkinsfile`.
- Install/configure AWS CLI, Docker Engine and `kubectl` on the Jenkins agent. The agent must be able to build Docker images.
- Prefer an attached IAM role for the Jenkins agent rather than long-lived AWS access keys.
- Give the agent IAM permissions for ECR repository/image operations and EKS cluster discovery. Configure EKS access so the pipeline identity can deploy into the `freshmart` namespace.
- Edit `AWS_REGION` and `EKS_CLUSTER` in the Jenkinsfile. Ensure the ECR repository name matches `ECR_REPOSITORY`.
- The pipeline tags images with Jenkins `BUILD_NUMBER`; ECR repo is created if missing.

## Important limitations
- This is a **frontend demo** only. The cart/order is browser-side demo state. It has no real database, authentication, stock management, order backend, or payment processing.
- The EKS cluster, ECR repository, IAM permissions, VPC/networking, and Jenkins server are not created by this package. Those must be provisioned/configured in your AWS account.
- A `LoadBalancer` Service provisions cloud networking and can cost money. Delete resources when done:
```bash
kubectl delete namespace freshmart
```
- For a real production site, prefer HTTPS with a domain, ingress/load balancer configuration, durable backend/database, secrets handling, and monitoring. Avoid `latest` tags; use immutable version tags as the pipeline does.
