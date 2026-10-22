
pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['BUILD_ONLY', 'DEPLOY'],
            description: 'BUILD_ONLY tests the image; DEPLOY publishes and deploys it'
        )
    }

    environment {
        IMAGE_NAME = 'sanjaidharan21/road-hazard-portal'
        IMAGE_TAG = 'v1'
        DEPLOYMENT_NAME = 'road-hazard-portal'
        CONTAINER_NAME = 'road-hazard-portal'
        SERVICE_NAME = 'road-hazard-service'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Parallel Code Checks') {
            parallel {
                stage('HTML File Check') {
                    steps {
                        bat 'if exist index.html (echo HTML file found) else (exit /b 1)'
                    }
                }

                stage('Dockerfile Check') {
                    steps {
                        bat 'if exist Dockerfile (echo Dockerfile found) else (exit /b 1)'
                    }
                }

                stage('Kubernetes Manifest Check') {
                    steps {
                        bat 'if exist deployment.yaml (echo deployment.yaml found) else (exit /b 1)'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% .'
            }
        }

        stage('Local Container Test') {
            steps {
                bat '''
                    docker rm -f road-hazard-ci-test 2>NUL
                    docker run -d --name road-hazard-ci-test -p 18085:80 %IMAGE_NAME%:%IMAGE_TAG%
                    curl.exe --fail http://localhost:18085
                    if errorlevel 1 exit /b 1
                    docker rm -f road-hazard-ci-test
                '''
            }
        }

        stage('Publish Docker Image') {
            when {
                expression { params.ACTION == 'DEPLOY' }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    bat '''
                        echo %DOCKER_TOKEN% | docker login -u %DOCKER_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push %IMAGE_NAME%:%IMAGE_TAG%
                        if errorlevel 1 exit /b 1
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { params.ACTION == 'DEPLOY' }
            }
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig-file',
                    variable: 'KUBECONFIG'
                )]) {
                    bat '''
                        kubectl --kubeconfig="%KUBECONFIG%" set image deployment/%DEPLOYMENT_NAME% %CONTAINER_NAME%=%IMAGE_NAME%:%IMAGE_TAG%
                        if errorlevel 1 exit /b 1
                        kubectl --kubeconfig="%KUBECONFIG%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f road-hazard-ci-test 2>NUL & exit /b 0'
        }
        success {
            echo 'Road Hazard Portal pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output for the failing stage.'
        }
    }
}
