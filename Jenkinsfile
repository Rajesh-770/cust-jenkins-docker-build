pipeline {
    agent any

    environment {
        IMAGE_NAME     = 'rajesh4113/cust-jenkins-docker-build'
        HOST_PORT      = '8082'
        CONTAINER_PORT = '8080'
    }

    options {
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                // Checkout the same repo that contains this Jenkinsfile
                checkout scm
            }
        }

        stage('Verify Git Tag') {
            steps {
                script {
                    def tag = ''
                    try {
                        // This will fail if the current commit does NOT have an exact tag
                        tag = bat(
                            script: 'git describe --tags --exact-match',
                            returnStdout: true
                        ).trim()
                    } catch (Exception e) {
                        error "No Git tag found on this commit. Create and push a tag (e.g. v1.0.0) to run this pipeline."
                    }

                    env.IMAGE_TAG = tag
                    echo "Running release pipeline for tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Tool name must match Global Tool Configuration
                    def scannerHome = tool 'SonarScanner'

                    // 'LocalSonar' must match SonarQube server name in Manage Jenkins → System
                    withSonarQubeEnv('LocalSonar') {
                        bat """
                            "${scannerHome}\\bin\\sonar-scanner.bat" ^
                              -Dsonar.projectKey=cust-flask ^
                              -Dsonar.sources=. ^
                              -Dsonar.host.url=%SONAR_HOST_URL% ^
                              -Dsonar.login=%SONAR_AUTH_TOKEN%
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
                bat """
                    docker build -t ${IMAGE_NAME}:${env.IMAGE_TAG} .
                """
            }
        }

        stage('Trivy Scan') {
            steps {
                echo "Scanning Docker image with Trivy..."
                bat """
                    trivy image --severity HIGH,CRITICAL --exit-code 0 ${IMAGE_NAME}:${env.IMAGE_TAG}
                """
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo "Pushing image to DockerHub..."
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    bat """
                        echo %DOCKER_PASS% | docker login -u "%DOCKER_USER%" --password-stdin
                        docker push ${IMAGE_NAME}:${env.IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy Container') {
            steps {
                echo "Deploying container from image ${IMAGE_NAME}:${env.IMAGE_TAG}..."
                bat """
                    docker rm -f cust-app 2>nul || echo No existing container
                    docker run -d --name cust-app -p ${HOST_PORT}:${CONTAINER_PORT} ${IMAGE_NAME}:${env.IMAGE_TAG}
                """
            }
        }

        stage('Cleanup') {
            steps {
                echo "Cleaning up unused Docker images..."
                bat """
                    docker image prune -af || echo Cleanup finished
                """
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline completed successfully for tag ${env.IMAGE_TAG}."
        }
        failure {
            echo "CI/CD pipeline failed. Please check the stage logs."
        }
    }
}
