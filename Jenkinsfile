


pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        DOCKER_IMAGE = 'archanajawahar14/university-lms'
        DOCKER_CREDENTIALS = 'dockerhub-jenkins'
        KUBE_CREDENTIALS = 'kubernetes-config'
        DEPLOYMENT_NAME = 'university-lms'
        CONTAINER_NAME = 'university-lms'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    docker build -t %DOCKER_IMAGE%:v%BUILD_NUMBER% .
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Test Image') {
            steps {
                bat '''
                    docker image inspect %DOCKER_IMAGE%:v%BUILD_NUMBER%
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-jenkins',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DOCKER_TOKEN% | docker login -u %DOCKER_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push %DOCKER_IMAGE%:v%BUILD_NUMBER%
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
                        kubectl --kubeconfig "%KUBECONFIG%" config current-context
                        if errorlevel 1 exit /b 1
                        kubectl --kubeconfig "%KUBECONFIG%" set image deployment/%DEPLOYMENT_NAME% %CONTAINER_NAME%=%DOCKER_IMAGE%:v%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1
                        kubectl --kubeconfig "%KUBECONFIG%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s
                        if errorlevel 1 (
                            kubectl --kubeconfig "%KUBECONFIG%" rollout undo deployment/%DEPLOYMENT_NAME%
                            exit /b 1
                        )
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'University LMS deployed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs and Kubernetes rollout status.'
        }
    }
}
