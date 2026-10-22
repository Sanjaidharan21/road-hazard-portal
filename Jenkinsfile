
pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['BUILD_ONLY', 'DEPLOY'],
            description: 'BUILD_ONLY builds and tests; DEPLOY also publishes and deploys'
        )
    }

    environment {
        IMAGE_NAME = 'sanjaidharan21/road-hazard-portal'
        IMAGE_TAG = 'v1'
        DEPLOYMENT_NAME = 'road-hazard-portal'
        CONTAINER_NAME = 'road-hazard-portal'

        DOCKER_CLI = 'C:\\Users\\Sanjay\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_CLI = 'C:\\Users\\Sanjay\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
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
                bat '"%DOCKER_CLI%" build -t %IMAGE_NAME%:%IMAGE_TAG% .'
            }
        }

        stage('Local Container Test') {
            steps {
                bat '''
                    "%DOCKER_CLI%" rm -f road-hazard-ci-test 2>NUL
                    "%DOCKER_CLI%" run -d --name road-hazard-ci-test -p 18085:80 %IMAGE_NAME%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1

                    timeout /t 3 /nobreak >NUL
                    curl.exe --fail http://localhost:18085
                    if errorlevel 1 (
                        "%DOCKER_CLI%" logs road-hazard-ci-test
                        exit /b 1
                    )

                    "%DOCKER_CLI%" rm -f road-hazard-ci-test
                '''
            }
        }

        stage('Publish Docker Image') {
            when {
                expression { params.ACTION == 'DEPLOY' }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    bat '''
                        echo %DOCKER_TOKEN% | "%DOCKER_CLI%" login -u %DOCKER_USER% --password-stdin
                        if errorlevel 1 exit /b 1

                        "%DOCKER_CLI%" push %IMAGE_NAME%:%IMAGE_TAG%
                        if errorlevel 1 (
                            "%DOCKER_CLI%" logout
                            exit /b 1
                        )

                        "%DOCKER_CLI%" logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { params.ACTION == 'DEPLOY' }
            }
            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: 'kubeconfig-file',
                            variable: 'KUBECONFIG'
                        )
                    ]) {
                        try {
                            bat '''
                                "%KUBECTL_CLI%" --kubeconfig="%KUBECONFIG%" set image deployment/%DEPLOYMENT_NAME% %CONTAINER_NAME%=%IMAGE_NAME%:%IMAGE_TAG%
                                if errorlevel 1 exit /b 1

                                "%KUBECTL_CLI%" --kubeconfig="%KUBECONFIG%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s
                                if errorlevel 1 exit /b 1
                            '''
                        } catch (err) {
                            echo 'Deployment failed. Attempting automatic rollback.'

                            bat(
                                returnStatus: true,
                                script: '"%KUBECTL_CLI%" --kubeconfig="%KUBECONFIG%" rollout undo deployment/%DEPLOYMENT_NAME%'
                            )

                            bat(
                                returnStatus: true,
                                script: '"%KUBECTL_CLI%" --kubeconfig="%KUBECONFIG%" rollout status deployment/%DEPLOYMENT_NAME% --timeout=120s'
                            )

                            throw err
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            bat '"%DOCKER_CLI%" rm -f road-hazard-ci-test 2>NUL & exit /b 0'
        }

        success {
            echo 'Road Hazard Portal pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Console Output for the failing stage and rollback result.'
        }
    }
}
