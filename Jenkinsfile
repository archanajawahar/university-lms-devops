
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'archanajawahar14/university-lms'
    }

    stages {
        stage('Checkout GitHub') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/archanajawahar/university-lms-devops.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    docker build -t %IMAGE_NAME%:v%BUILD_NUMBER% .
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                bat '''
                    docker image inspect %IMAGE_NAME%:v%BUILD_NUMBER%
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Login and Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-lms-new',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    bat '''
                        @echo off
                        docker login --username "%DH_USER%" --password-stdin < nul
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-lms-new',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DH_TOKEN% | docker login --username "%DH_USER%" --password-stdin
                        if errorlevel 1 exit /b 1

                        docker push %IMAGE_NAME%:v%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat '''
                        @echo off

                        kubectl --kubeconfig "%KUBECONFIG%" set image deployment/university-lms university-lms=%IMAGE_NAME%:v%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig "%KUBECONFIG%" rollout status deployment/university-lms --timeout=120s
                        if errorlevel 1 (
                            kubectl --kubeconfig "%KUBECONFIG%" rollout undo deployment/university-lms
                            exit /b 1
                        )
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'University LMS pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output for the failed stage.'
        }
    }
}
