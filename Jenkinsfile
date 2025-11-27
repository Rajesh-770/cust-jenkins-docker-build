pipeline {

    agent any

    environment {
        DOCKER_IMAGE  = "rajesh4113/cust-app"
        SONAR_PROJECT = "cust-flask"
        // RELEASE_TAG will be set in Verify Git Tag stage
    }

    stages {

        /* --- 1. Checkout from GitHub (done automatically, but keep it) --- */
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        /* --- 2. Only run for tagged builds (v1.x.x etc.) --- */
        stage('Verify Git Tag') {
            steps {
                script {
                    // Make sure we have all tags
                    bat 'git fetch --tags --force'

                    // Try to get tag for current commit; if none, echo NOT_TAGGED
                    def tagCheck = bat(
                        script: 'git describe --tags --exact-match HEAD 2>NUL || echo NOT_TAGGED',
                        returnStdout: true
                    ).trim()

                    if (tagCheck == 'NOT_TAGGED' || !tagCheck.startsWith("v")) {
                        error "❌ Not a tagged build. CI/CD runs only on tagged releases."
                    }

                    env.RELEASE_TAG = tagCheck
                    echo "✔ Running pipeline for release tag: ${env.RELEASE_TAG}"
                }
            }
        }

        /* --- 3. SonarQube static analysis --- */
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'      // Jenkins global tool name
                    withSonarQubeEnv('SonarScanner') {          // Jenkins Sonar server name
                        bat """
                            "${scannerHome}\\bin\\sonar-scanner.bat" ^
                              -Dsonar.projectKey=${SONAR_PROJECT} ^
                              -Dsonar.sources=. ^
                              -Dsonar.projectVersion=${env.RELEASE_TAG}
                        """
                    }
                }
            }
        }

        /* --- 4. Build Docker image --- */
        stage('Build Docker Image') {
            steps {
                script {
                    bat """
                        docker build -t ${DOCKER_IMAGE}:${env.RELEASE_TAG} .
                    """
                }
            }
        }

        /* --- 5. Trivy image scan (fail on CRITICAL vulns) --- */
        stage('Trivy Scan') {
            steps {
                script {
                    bat """
                        trivy image --severity CRITICAL --exit-code 1 ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                    """
                }
            }
        }

        /* --- 6. Push to DockerHub (using dockerhub-creds) --- */
        stage('Push to DockerHub') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        bat """
                            docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                            docker push ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                            docker logout
                        """
                    }
                }
            }
        }

        /* --- 7. Deploy to Minikube --- */
        stage('Deploy to Minikube') {
            steps {
                script {
                    bat """
                        rem Load the tagged image into Minikube
                        minikube image load ${DOCKER_IMAGE}:${env.RELEASE_TAG}

                        rem Apply (or create) deployment and service
                        kubectl apply -f k8s-deployment.yaml
                        kubectl apply -f k8s-service.yaml

                        rem Force deployment to use the correct tagged image
                        kubectl set image deployment/cust-app cust-app=${DOCKER_IMAGE}:${env.RELEASE_TAG} --record
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 CI/CD Pipeline SUCCESS — deployed version: ${env.RELEASE_TAG}"
        }
        failure {
            echo "❌ CI/CD pipeline FAILED — check stage logs."
        }
    }
}
