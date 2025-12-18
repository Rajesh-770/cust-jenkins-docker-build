pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        ECR_REGISTRY = "423014875134.dkr.ecr.ap-south-1.amazonaws.com"
        ECR_REPO = "cust-jenkins-docker-build"
        IMAGE_TAG = "latest"
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${ECR_REPO}:${IMAGE_TAG} .
                """
            }
        }

        stage('Login to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_REGION} \
                | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                """
            }
        }

        stage('Tag & Push Image to ECR') {
            steps {
                sh """
                docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh """
                kubectl apply -f k8s-deployment.yaml
                kubectl apply -f k8s-service.yaml
                """
            }
        }
    }

    post {
        success {
            echo "✅ CI/CD SUCCESS — App Deployed to Minikube"
        }
        failure {
            echo "❌ CI/CD FAILED — Check logs"
        }
    }
}
