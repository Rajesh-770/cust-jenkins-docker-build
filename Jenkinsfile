pipeline {

    agent any

    environment {
        DOCKER_IMAGE  = "rajesh4113/cust-app"
        SONAR_PROJECT = "cust-flask"      // must match SonarQube project key
        // RELEASE_TAG will be set dynamically from git tag or fallback
    }

    stages {

        /* --- 1. Checkout from GitHub --- */
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        /* --- 2. Tag / version handling --- */
        stage('Verify Git Tag') {
            steps {
                script {
                    // Ensure all tags are available
                    bat 'git fetch --tags --force'

                    // Get ONLY the tag(s) pointing at HEAD (Windows-safe)
                    def tagOutput = bat(
                        script: '@echo off && for /f "usebackq delims=" %%i in (`git tag --points-at HEAD`) do @echo %%i',
                        returnStdout: true
                    ).trim()

                    if (!tagOutput) {
                        // no tag -> use a dev-style version
                        env.RELEASE_TAG = "dev-${env.BUILD_NUMBER}"
                        echo "No git tag found, using fallback release tag: ${env.RELEASE_TAG}"
                    } else {
                        // If multiple tags, pick the first one
                        def tagCheck = tagOutput.split()[0]

                        if (!tagCheck.startsWith("v")) {
                            error "Tag '${tagCheck}' does not start with 'v'. Use tags like v1.0.12."
                        }

                        env.RELEASE_TAG = tagCheck
                        echo "Running pipeline for release tag: ${env.RELEASE_TAG}"
                    }
                }
            }
        }

        /* --- 3. SonarQube static analysis --- */
        stage('SonarQube Analysis') {
            steps {
                script {
                    // Jenkins tool name from: Manage Jenkins → Global Tool Configuration
                    def scannerHome = tool 'SonarScanner'

                    withCredentials([
                        string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')
                    ]) {
                        bat """
                            "${scannerHome}\\bin\\sonar-scanner.bat" ^
                              -Dsonar.host.url=http://localhost:9000 ^
                              -Dsonar.projectKey=${SONAR_PROJECT} ^
                              -Dsonar.sources=. ^
                              -Dsonar.projectVersion=${env.RELEASE_TAG} ^
                              -Dsonar.token=%SONAR_TOKEN%
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

        /* --- 5. Trivy image scan --- */
        stage('Trivy Scan') {
            steps {
                script {
                    bat """
                        trivy image --severity CRITICAL --exit-code 1 ${DOCKER_IMAGE}:${env.RELEASE_TAG}
                    """
                }
            }
        }

        /* --- 6. Push image to DockerHub --- */
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

        /* --- 7. Deploy to Minikube (using kubectl + shared kubeconfig) --- */
        stage('Deploy to Minikube') {
            steps {
                script {
                    bat """
                        echo ==== Configure kubeconfig for Jenkins ====
                        set KUBECONFIG=C:\\ProgramData\\Jenkins\\.kube\\config

                        echo ==== Check cluster access ====
                        kubectl config current-context
                        kubectl get nodes

                        echo ==== Apply Kubernetes manifests ====
                        kubectl apply -f k8s-deployment.yaml --validate=false
                        kubectl apply -f k8s-service.yaml --validate=false

                        echo ==== Update deployment image ====
                        kubectl set image deployment/cust-app cust-app=${DOCKER_IMAGE}:${env.RELEASE_TAG}

                        echo ==== Wait for rollout ====
                        kubectl rollout status deployment/cust-app --timeout=120s
                    """
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline SUCCESS — deployed version: ${env.RELEASE_TAG}"
        }
        failure {
            echo "CI/CD pipeline FAILED — check stage logs."
        }
    }
}
