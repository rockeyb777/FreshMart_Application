pipeline {
  agent any
  options { timestamps(); disableConcurrentBuilds() }
  environment {
    AWS_REGION = 'ap-south-1'
    EKS_CLUSTER = 'REPLACE_WITH_EKS_CLUSTER_NAME'
    ECR_REPOSITORY = 'freshmart-web'
    APP_NAMESPACE = 'freshmart'
    APP_DEPLOYMENT = 'freshmart-web'
    CONTAINER_NAME = 'web'
  }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Validate files') {
      steps {
        sh '''
          test -s index.html
          test -s Dockerfile
          test -s k8s/deployment.yaml
        '''
      }
    }
    stage('Build image') {
      steps {
        sh '''
          set -eu
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          REGISTRY="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          IMAGE_URI="$REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER"
          echo "$IMAGE_URI" > image-uri.txt
          docker build --pull -t "$IMAGE_URI" .
        '''
      }
    }
    stage('Push to ECR') {
      steps {
        sh '''
          set -eu
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          REGISTRY="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          aws ecr describe-repositories --repository-names "$ECR_REPOSITORY" --region "$AWS_REGION" >/dev/null 2>&1 || \
            aws ecr create-repository --repository-name "$ECR_REPOSITORY" --image-scanning-configuration scanOnPush=true --region "$AWS_REGION"
          aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$REGISTRY"
          docker push "$(cat image-uri.txt)"
        '''
      }
    }
    stage('Deploy to EKS') {
      steps {
        sh '''
          set -eu
          aws eks update-kubeconfig --name "$EKS_CLUSTER" --region "$AWS_REGION"
          kubectl apply -f k8s/namespace.yaml
          kubectl apply -f k8s/service.yaml
          kubectl apply -f k8s/deployment.yaml
          kubectl set image deployment/$APP_DEPLOYMENT $CONTAINER_NAME="$(cat image-uri.txt)" -n "$APP_NAMESPACE"
          kubectl rollout status deployment/$APP_DEPLOYMENT -n "$APP_NAMESPACE" --timeout=180s
          kubectl get pods,svc -n "$APP_NAMESPACE"
        '''
      }
    }
  }
  post {
    always { sh 'docker logout || true' }
    success { echo 'FreshMart image published and EKS rollout completed.' }
    failure { echo 'Pipeline failed. Check stage logs and Kubernetes events.' }
  }
}
