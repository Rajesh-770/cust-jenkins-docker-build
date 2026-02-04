pipeline {
    agent any

    environment {
        APP_NAME = "cust-flask"
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Verify Git Tag') {
            steps {
                script {
                    if (!env.GIT_TAG_NAME) {
                        error "Pipeline must be triggered by a Git tag"
                    }
                    echo "Running CI/CD for tag: ${env.GIT_TAG_NAME}"
                }
            }
        }

        stage('Use Minikube Docker') {
            steps {
                sh '''
                eval $(minikube docker-env)
                '''
            }
        }

        stage('Build Docker Image (Local)') {
            steps {
                sh """
                docker build -t ${APP_NAME}:${env.GIT_TAG_NAME} .
                """
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh """
                sed -i 's/IMAGE_TAG/${env.GIT_TAG_NAME}/g' k8s-deployment.yaml
                kubectl apply -f k8s-deployment.yaml
                kubectl apply -f k8s-service.yaml
                """
            }
        }
    }

    post {
        success {
            echo "✅ CI/CD SUCCESS — App deployed to Minikube with tag ${env.GIT_TAG_NAME}"
        }
        failure {
            echo "❌ CI/CD FAILED — Check logs"
        }
    }
}
