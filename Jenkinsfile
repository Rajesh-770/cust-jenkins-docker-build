pipeline {

    agent any

    environment {
        DOCKER_IMAGE = "rajesh4113/cust-app"
        SONAR_PROJECT = "cust-flask"
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
                    def tagCheck = bat(script: 'git describe --tags --exact-match', returnStdout: true).trim()

                    if (!tagCheck.startsWith("v")) {
                        error "❌ Not a tagged build. CI/CD only runs for tagged releases."
                    }

                    echo "Running pipeline for release tag: ${tagCheck}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarScanner') {  // Must match the server name in Jenkins config
                        bat """
                            "${scannerHome}\\bin\\sonar-scanner.bat" ^
                              -Dsonar.projectKey=${SONAR_PROJECT} ^
                              -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    bat """
                        docker build -t ${DOCKER_IMAGE}:${BUILD_TAG} .
                    """
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    bat """
                        trivy image --severity CRITICAL ${DOCKER_IMAGE}:${BUILD_TAG}
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-cred',
                     usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {

                        bat """
                            docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                            docker push ${DOCKER_IMAGE}:${BUILD_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                script {
                    bat """
                        minikube image rm ${DOCKER_IMAGE}:${BUILD_TAG} || exit 0
                        minikube image load ${DOCKER_IMAGE}:${BUILD_TAG}
                        kubectl apply -f k8s-deployment.yaml
                        kubectl apply -f k8s-service.yaml
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Build and deployment successful."
        }
        failure {
            echo "CI/CD pipeline failed. Check logs."
        }
    }
}
