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
                    // Ensure Jenkins pulls all remote tags
                    bat """
                        git fetch --tags
                    """

                    // Detect the release tag
                    def tagCheck = bat(script: 'git describe --tags --exact-match', returnStdout: true).trim()

                    if (!tagCheck.startsWith("v")) {
                        error "❌ Not a tagged build. CI/CD only runs for tagged releases."
                    }

                    env.RELEASE_TAG = tagCheck
                    echo "📌 Running pipeline for release tag: ${RELEASE_TAG}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarScanner') {
                        bat """
                            "${scannerHome}\\bin\\sonar-scanner.bat" ^
                             -Dsonar.projectKey=${SONAR_PROJECT} ^
                             -Dsonar.sources=. ^
                             -Dsonar.projectVersion=${RELEASE_TAG}
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    bat """
                        docker build -t ${DOCKER_IMAGE}:${RELEASE_TAG} .
                    """
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    bat """
                        trivy image --severity CRITICAL ${DOCKER_IMAGE}:${RELEASE_TAG}
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {

                        bat """
                            docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                            docker push ${DOCKER_IMAGE}:${RELEASE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                script {
                    bat """
                        minikube image rm ${DOCKER_IMAGE}:${RELEASE_TAG} || exit 0
                        minikube image load ${DOCKER_IMAGE}:${RELEASE_TAG}
                        kubectl apply -f k8s-deployment.yaml
                        kubectl apply -f k8s-service.yaml
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 CI/CD Pipeline Success — deployed version: ${RELEASE_TAG}"
        }
        failure {
            echo "❌ CI/CD pipeline failed — check logs."
        }
    }
}
