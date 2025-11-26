pipeline {

    agent any

    environment {
        DOCKER_IMAGE        = "rajesh4113/cust-app"      // DockerHub repo
        SONAR_PROJECT       = "cust-flask"              // SonarQube project key
        SONARQUBE_SERVER    = "SonarScanner"            // Must match Manage Jenkins -> SonarQube servers
        SONAR_SCANNER_TOOL  = "SonarScanner"            // Must match Manage Jenkins -> Global Tool Configuration
        DOCKER_CRED_ID      = "docker-cred"             // Jenkins credential ID for DockerHub
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
                    // Try to get an exact tag for this commit. If none, return NO_TAG instead of failing the step.
                    def tagCheck = bat(
                        script: 'git describe --tags --exact-match 2>nul || echo NO_TAG',
                        returnStdout: true
                    ).trim()

                    if (tagCheck == 'NO_TAG' || !tagCheck.startsWith("v")) {
                        error "Not a tagged build. CI/CD only runs for tagged releases."
                    }

                    env.RELEASE_TAG = tagCheck
                    echo "Running pipeline for release tag: ${env.RELEASE_TAG}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool SONAR_SCANNER_TOOL

                    withSonarQubeEnv(SONARQUBE_SERVER) {
                        bat """
                            "${scannerHome}\\bin\\sonar-scanner.bat" ^
                              -Dsonar.projectKey=${SONAR_PROJECT} ^
                              -Dsonar.sources=. ^
                              -Dsonar.host.url=%SONAR_HOST_URL%
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Use the Git tag as Docker tag (ex: v1.0.2)
                    bat "docker build -t ${DOCKER_IMAGE}:${env.RELEASE_TAG} ."
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    // Scan the built image for HIGH and CRITICAL vulnerabilities
                    bat """
                        trivy image --severity HIGH,CRITICAL --exit-code 1 ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: DOCKER_CRED_ID,
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        bat """
                            echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
                            docker push ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                script {
                    // Assumes kubectl context is set to Minikube and YAML files exist in the repo
                    bat """
                        minikube image rm ${DOCKER_IMAGE}:${env.RELEASE_TAG} 2>nul || echo No existing local image
                        minikube image load ${DOCKER_IMAGE}:${env.RELEASE_TAG}
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
