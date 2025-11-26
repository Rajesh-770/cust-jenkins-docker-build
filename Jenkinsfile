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
                    // Get tags pointing at current HEAD
                    def output = bat(
                        script: 'git tag --points-at HEAD',
                        returnStdout: true
                    ).trim()

                    def lines = output.readLines()
                    def tag = lines ? lines[-1].trim() : ""

                    if (!tag || !tag.startsWith("v")) {
                        error "Not a tagged build. CI/CD only runs for tagged releases."
                    }

                    // Save the tag into env so we can reuse it
                    env.RELEASE_TAG = tag
                    echo "Running pipeline for release tag: ${tag}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Jenkins tool name: SonarScanner
                    def scannerHome = tool 'SonarScanner'

                    // Jenkins SonarQube server name: SonarScanner
                    withSonarQubeEnv('SonarScanner') {
                        bat """
                            "${scannerHome}\\bin\\sonar-scanner.bat" ^
                              -Dsonar.projectKey=${SONAR_PROJECT} ^
                              -Dsonar.sources=. ^
                              -Dsonar.host.url=http://localhost:9000
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat """
                    docker build -t ${DOCKER_IMAGE}:${env.RELEASE_TAG} .
                """
            }
        }

        stage('Trivy Scan') {
            steps {
                bat """
                    trivy image --severity HIGH,CRITICAL --exit-code 0 ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                """
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
                            echo Logging into DockerHub...
                            echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
                            docker push ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                bat """
                    minikube image load ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                    kubectl apply -f k8s-deployment.yaml
                    kubectl apply -f k8s-service.yaml
                """
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
