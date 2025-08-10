pipeline {
    agent any
    stages {
        stage('Clone repo') {
            steps {
                git branch: 'main', url: 'https://github.com/Rajesh-770/cust-jenkins-docker-build.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("custom-jenkins-docker-build:latest")
                }
            }
        }
        stage('List Docker Images') {
            steps {
                sh 'docker images'
            }
        }
    }
}
