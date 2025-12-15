pipeline {
    agent any

    environment {
        AWS_REGION     = "ap-south-1"
        AWS_ACCOUNT_ID = "423014875134"
        ECR_REPO       = "cust-jenkins-docker-build"
        ECR_IMAGE      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"

        SONAR_PROJECT  = "cust-flask"
        MINIKUBE_HOST  = "10.0.13.114"
    }

    stages {

        /* 1️⃣ Checkout */
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        /* 2️⃣ Versioning */
        stage('Set Release Tag') {
            steps {
                script {
                    env.RELEASE_TAG = "dev-${BUILD_NUMBER}"
                    echo "Release tag: ${RELEASE_TAG}"
                }
            }
        }

        /* 3️⃣ SonarQube Scan */
        stage('SonarQube Analysis') {
            steps {
                withCredentials([
                    string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')
                ]) {
                    sh """
                      sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT} \
                        -Dsonar.sources=. \
                        -Dsonar.projectVersion=${RELEASE_TAG} \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        /* 4️⃣ Quality Gate (REAL CI/CD) */
        stage('Sonar Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* 5️⃣ Login to ECR (IAM Role) */
        stage('Login to ECR') {
            steps {
                sh """
                  aws ecr get-login-password --region ${AWS_REGION} |
                  docker login --username AWS --password-stdin \
                  ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                """
            }
        }

        /* 6️⃣ Build Docker Image */
        stage('Build Docker Image') {
            steps {
                sh """
                  docker build -t ${ECR_IMAGE}:${RELEASE_TAG} .
                """
            }
        }

        /* 7️⃣ Trivy Scan */
        stage('Trivy Scan') {
            steps {
                sh """
                  trivy image --severity CRITICAL --exit-code 1 \
                  ${ECR_IMAGE}:${RELEASE_TAG}
                """
            }
        }

        /* 8️⃣ Push to ECR */
        stage('Push Image to ECR') {
            steps {
                sh """
                  docker push ${ECR_IMAGE}:${RELEASE_TAG}
                """
            }
        }

        /* 9️⃣ Deploy to Minikube (REMOTE EC2) */
        stage('Deploy to Minikube') {
            steps {
                sshagent(['minikube-ssh-key']) {
                    sh """
                      ssh -o StrictHostKeyChecking=no ubuntu@${MINIKUBE_HOST} '
                        kubectl apply -f k8s-deployment.yaml
                        kubectl apply -f k8s-service.yaml
                        kubectl set image deployment/cust-app \
                          cust-app=${ECR_IMAGE}:${RELEASE_TAG}
                        kubectl rollout status deployment/cust-app
                      '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ CI/CD SUCCESS — ${RELEASE_TAG} deployed"
        }
        failure {
            echo "❌ CI/CD FAILED — check logs"
        }
    }
}
