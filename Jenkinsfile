
pipeline {
    agent any

    parameters {
        string(name: 'VERSION', defaultValue: 'v1.0',
               description: 'Release version')
        booleanParam(name: 'SIMULATE_FAILURE', defaultValue: false,
                     description: 'Demonstrate automatic rollback')
    }

    environment {
        IMAGE = 'arshiaanjumd/food-ordering-portal'
        DOCKER_EXE = 'C:\\Users\\arshi\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_EXE = 'C:\\Users\\arshi\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                bat '"%DOCKER_EXE%" build -t %IMAGE%:%VERSION% .'
            }
        }

        stage('Automated Test') {
            steps {
                bat '"%DOCKER_EXE%" run --rm %IMAGE%:%VERSION% nginx -t'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    bat '''
                        @echo off
                        echo %DH_TOKEN%| "%DOCKER_EXE%" login -u %DH_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        "%DOCKER_EXE%" push %IMAGE%:%VERSION%
                        set RESULT=%ERRORLEVEL%
                        "%DOCKER_EXE%" logout
                        exit /b %RESULT%
                    '''
                }
            }
        }

        stage('Blue-Green Deploy and Verify') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    script {
                        def previousImage = bat(
                            script: '@echo off & "%KUBECTL_EXE%" get deployment food-ordering -o=jsonpath="{.spec.template.spec.containers[0].image}"',
                            returnStdout: true
                        ).trim()

                        try {
                            if (params.SIMULATE_FAILURE) {
                                bat '"%KUBECTL_EXE%" set image deployment/food-ordering food-ordering=%IMAGE%:%VERSION%-broken'
                            } else {
                                bat '"%KUBECTL_EXE%" set image deployment/food-ordering food-ordering=%IMAGE%:%VERSION%'
                            }

                            bat '''
                                "%KUBECTL_EXE%" rollout status deployment/food-ordering --timeout=120s
                                if errorlevel 1 exit /b 1
                                "%KUBECTL_EXE%" get deployment food-ordering
                                "%KUBECTL_EXE%" get pods -l app=food-ordering
                                "%KUBECTL_EXE%" get service food-ordering-service
                            '''
                        } catch (err) {
                            echo "Failed release. Previous image: ${previousImage}"
                            bat '''
                                "%KUBECTL_EXE%" rollout undo deployment/food-ordering
                                if errorlevel 1 exit /b 1
                                "%KUBECTL_EXE%" rollout status deployment/food-ordering --timeout=120s
                                if errorlevel 1 exit /b 1
                                "%KUBECTL_EXE%" get pods -l app=food-ordering
                            '''
                            error('Release failed; automatic rollback attempted.')
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Food Ordering CI/CD pipeline completed.'
        }
    }
}
