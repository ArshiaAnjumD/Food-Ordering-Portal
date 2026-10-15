
pipeline {
    agent any

    parameters {
        string(
            name: 'VERSION',
            defaultValue: 'v1.0',
            description: 'Release version, e.g. v1.0 or v2.0'
        )
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: 'Demonstrate automatic rollback'
        )
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

        stage('Validate Version') {
            steps {
                script {
                    if (!params.VERSION?.trim()) {
                        env.RELEASE_VERSION = 'v1.0'
                    } else {
                        env.RELEASE_VERSION = params.VERSION.trim()
                    }

                    if (!(env.RELEASE_VERSION ==~ /[A-Za-z0-9][A-Za-z0-9_.-]*/)) {
                        error('Invalid VERSION. Use a value such as v1.0 or v2.0.')
                    }
                }
                echo "Building release ${env.RELEASE_VERSION}"
            }
        }

        stage('Build') {
            steps {
                bat '''
                    "%DOCKER_EXE%" build -t %IMAGE%:%RELEASE_VERSION% .
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Automated Test') {
            steps {
                bat '''
                    "%DOCKER_EXE%" run --rm %IMAGE%:%RELEASE_VERSION% nginx -t
                    if errorlevel 1 exit /b 1
                '''
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

                        "%DOCKER_EXE%" push %IMAGE%:%RELEASE_VERSION%
                        set PUSH_RESULT=%ERRORLEVEL%

                        "%DOCKER_EXE%" logout
                        exit /b %PUSH_RESULT%
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
                        def oldImage = bat(
                            script: '@echo off & "%KUBECTL_EXE%" get deployment food-ordering -o=jsonpath="{.spec.template.spec.containers[0].image}"',
                            returnStdout: true
                        ).trim()

                        if (!oldImage) {
                            error('Could not read the current deployment image.')
                        }

                        try {
                            if (params.SIMULATE_FAILURE) {
                                bat '''
                                    "%KUBECTL_EXE%" set image deployment/food-ordering food-ordering=%IMAGE%:%RELEASE_VERSION%-broken
                                    if errorlevel 1 exit /b 1
                                '''
                            } else {
                                bat '''
                                    "%KUBECTL_EXE%" set image deployment/food-ordering food-ordering=%IMAGE%:%RELEASE_VERSION%
                                    if errorlevel 1 exit /b 1
                                '''
                            }

                            bat '''
                                "%KUBECTL_EXE%" rollout status deployment/food-ordering --timeout=120s
                                if errorlevel 1 exit /b 1

                                "%KUBECTL_EXE%" get deployment food-ordering
                                "%KUBECTL_EXE%" get pods -l app=food-ordering
                                "%KUBECTL_EXE%" get service food-ordering-service
                            '''

                            echo 'Deployment rollout and verification succeeded.'

                        } catch (err) {
                            echo "Release failed. Attempting rollback to previous revision: ${oldImage}"

                            bat '''
                                "%KUBECTL_EXE%" rollout undo deployment/food-ordering
                                if errorlevel 1 exit /b 1

                                "%KUBECTL_EXE%" rollout status deployment/food-ordering --timeout=120s
                                if errorlevel 1 exit /b 1

                                "%KUBECTL_EXE%" get pods -l app=food-ordering
                            '''

                            error('Release failed. Rollback was attempted.')
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Food Ordering Portal pipeline finished.'
        }
    }
}
