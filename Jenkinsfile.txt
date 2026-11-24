pipeline {
    agent any
    options { timestamps() }

    environment {
        IMAGE_SERVER = 'imendimassi/mern-server'
        IMAGE_CLIENT = 'imendimassi/mern-client'
        TAG          = "build-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Docker Build') {
            steps {
                bat 'docker version'
                bat "docker build -t %IMAGE_SERVER%:%TAG% ./Server"
                bat "docker build -t %IMAGE_CLIENT%:%TAG% ./Client"
            }
        }

        stage('Deploy (Docker Compose)') {
            steps {
                bat 'docker compose down'
                bat 'docker compose up -d --build'
            }
        }

        stage('Smoke Test') {
            steps {
                bat """
                ping -n 8 127.0.0.1 > nul
                curl -I http://localhost:3000 | find "200 OK"
                docker ps
                """
            }
        }

        stage('Push (Docker Hub)') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'USER',
                                                  passwordVariable: 'PASS')]) {
                    bat """
echo %PASS%| docker login -u %USER% --password-stdin
                    docker tag %IMAGE_SERVER%:%TAG% %IMAGE_SERVER%:latest
                    docker tag %IMAGE_CLIENT%:%TAG% %IMAGE_CLIENT%:latest
                    docker push %IMAGE_SERVER%:%TAG%
                    docker push %IMAGE_SERVER%:latest
                    docker push %IMAGE_CLIENT%:%TAG%
                    docker push %IMAGE_CLIENT%:latest
                    """
                }
            }
        }
    }

    post {
        success { echo 'Build+Deploy+Test+Push OK' }
        failure { echo 'Build/Deploy/Tests/Push KO' }
    }
}