pipeline {
    agent any

    environment {
        APP_NAME = "cust-flask"
        IMAGE_TAG = "latest"
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Use Minikube Docker') {
            steps {
                powershell '''
                & minikube docker-env --shell powershell | Invoke-Expression
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                docker build -t cust-flask:latest .
                '''
            }
        }

        stage('Deploy to Minikube') {
            steps {
                powershell '''
                kubectl apply -f k8s-deployment.yaml
                kubectl apply -f k8s-service.yaml
                '''
            }
        }
    }

    post {
        success {
            echo "CI/CD SUCCESS — App deployed to Minikube"
        }
        failure {
            echo "CI/CD FAILED — Check logs"
        }
    }
}
