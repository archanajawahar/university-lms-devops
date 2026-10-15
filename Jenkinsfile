
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'archanajawahar14/university-lms'
        K8S_DEPLOYMENT = 'university-lms'
        K8S_CONTAINER = 'university-lms'
        K8S_SERVICE = 'university-lms-service'
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
                    @echo off
                    docker build -t %IMAGE_NAME%:v%BUILD_NUMBER% .
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                bat '''
                    @echo off
                    docker image inspect %IMAGE_NAME%:v%BUILD_NUMBER% > nul
                    if errorlevel 1 exit /b 1
                    echo Docker image inspection passed.
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
                        powershell -NoProfile -NonInteractive -ExecutionPolicy Bypass -Command "$env:DH_TOKEN | docker login --username $env:DH_USER --password-stdin"
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

                        kubectl --kubeconfig "%KUBECONFIG%" set image deployment/%K8S_DEPLOYMENT% %K8S_CONTAINER%=%IMAGE_NAME%:v%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig "%KUBECONFIG%" rollout status deployment/%K8S_DEPLOYMENT% --timeout=120s

                        if errorlevel 1 (
                            echo Rollout failed. Attempting rollback...
                            kubectl --kubeconfig "%KUBECONFIG%" rollout undo deployment/%K8S_DEPLOYMENT%
                            kubectl --kubeconfig "%KUBECONFIG%" rollout status deployment/%K8S_DEPLOYMENT% --timeout=120s
                            exit /b 1
                        )

                        echo Kubernetes rollout completed successfully.
                    '''
                }
            }
        }

        stage('Verify Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat '''
                        @echo off

                        kubectl --kubeconfig "%KUBECONFIG%" get deployment %K8S_DEPLOYMENT%
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig "%KUBECONFIG%" get pods -l app=university-lms
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig "%KUBECONFIG%" get service %K8S_SERVICE%
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: University LMS image pushed and Kubernetes deployment verified.'
        }

        failure {
            echo 'FAILURE: Check the failed stage in Console Output.'
        }

        always {
            echo 'University LMS pipeline finished.'
        }
    }
}
