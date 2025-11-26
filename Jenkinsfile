pipeline {

    agent any

    environment {
        DOCKER_IMAGE  = "rajesh4113/cust-app"
        SONAR_PROJECT = "cust-flask"
        KUBE_NAMESPACE = "default"
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
                    // Make sure tags are available in this workspace
                    bat 'git fetch --tags'

                    // First check if this commit has an exact tag
                    int status = bat(
                        script: 'git describe --tags --exact-match 1>tag.txt 2>nul',
                        returnStatus: true
                    )

                    if (status != 0) {
                        error "Not a tagged build. CI/CD runs only for tagged releases."
                    }

                    // Read the tag value from the file created above
                    def tagValue = readFile('tag.txt').trim()
                    env.RELEASE_TAG = tagValue

                    echo "Running CI/CD pipeline for release tag: ${env.RELEASE_TAG}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Jenkins: Manage Jenkins -> Global Tool Configuration -> SonarScanner (name: SonarScanner)
                    def scannerHome = tool 'SonarScanner'

                    // Jenkins: Manage Jenkins -> System -> SonarQube servers (name: SonarScanner)
                    withSonarQubeEnv('SonarScanner') {
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

        stage('Build Docker Image') {
            steps {
                script {
                    bat """
                        docker build -t ${DOCKER_IMAGE}:${env.RELEASE_TAG} .
                    """
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    // --exit-code 0 = do not fail build on vulnerabilities (just report)
                    bat """
                        trivy image --severity HIGH,CRITICAL --exit-code 0 ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',   // from your screenshot
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        bat """
                            docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                            docker push ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                script {
                    bat """
                        minikube status

                        rem Remove old image inside Minikube (ignore error)
                        minikube image rm ${DOCKER_IMAGE}:${env.RELEASE_TAG} 2>nul || echo No old image

                        rem Load new image into Minikube Docker
                        minikube image load ${DOCKER_IMAGE}:${env.RELEASE_TAG}

                        rem Apply Kubernetes manifests
                        kubectl apply -n ${KUBE_NAMESPACE} -f k8s-deployment.yaml
                        kubectl apply -n ${KUBE_NAMESPACE} -f k8s-service.yaml

                        rem Show pods and services for confirmation
                        kubectl get pods -n ${KUBE_NAMESPACE}
                        kubectl get svc  -n ${KUBE_NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline completed successfully for tag ${env.RELEASE_TAG}."
        }
        failure {
            echo "CI/CD pipeline failed. Please inspect stage logs."
        }
    }
}
