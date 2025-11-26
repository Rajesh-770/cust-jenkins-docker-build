pipeline {

    agent any

    environment {
        DOCKER_IMAGE  = "rajesh4113/cust-app"
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
                    // Ensure tags are available in Jenkins workspace
                    bat 'git fetch --tags'

                    // Get the tag for this commit (last line of output)
                    def rawOutput = bat(
                        script: 'git describe --tags --exact-match',
                        returnStdout: true
                    ).trim()

                    def tagLines = rawOutput.readLines()
                    def tagCheck = tagLines[-1].trim()   // last line should be the tag, e.g. v1.0.4

                    if (!tagCheck || !tagCheck.startsWith("v")) {
                        error "Not a tagged build. CI/CD runs only for tagged releases."
                    }

                    env.RELEASE_TAG = tagCheck
                    echo "Running pipeline for release tag: ${env.RELEASE_TAG}"
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
                              -Dsonar.projectKey=${env.SONAR_PROJECT} ^
                              -Dsonar.sources=. ^
                              -Dsonar.projectVersion=${env.RELEASE_TAG}
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    bat """
                        docker build -t ${env.DOCKER_IMAGE}:${env.RELEASE_TAG} .
                    """
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    bat """
                        trivy image --severity CRITICAL ${env.DOCKER_IMAGE}:${env.RELEASE_TAG}
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-cred',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        bat """
                            docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                            docker push ${env.DOCKER_IMAGE}:${env.RELEASE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                script {
                    bat """
                        minikube image rm ${env.DOCKER_IMAGE}:${env.RELEASE_TAG} || exit 0
                        minikube image load ${env.DOCKER_IMAGE}:${env.RELEASE_TAG}
                        kubectl apply -f k8s-deployment.yaml
                        kubectl apply -f k8s-service.yaml
                    """
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline success — deployed version: ${env.RELEASE_TAG}"
        }
        failure {
            echo "CI/CD pipeline failed — check logs."
        }
    }
}
