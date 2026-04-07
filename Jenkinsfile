pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipStagesAfterUnstable()
    }

    environment {
        // Docker Hub: создайте credentials в Jenkins с ID docker-credentials (username + token).
        DOCKER_IMAGE_NAME = 'danyamki/mlops-team21'
        DOCKER_IMAGE_TAG  = "${env.BUILD_NUMBER}"
        DOCKER_REGISTRY   = 'docker.io'
        DOCKER_CREDENTIALS_ID = 'docker-credentials'

        GDRIVE_CREDENTIALS_ID = 'gdrive-sa'
        // Локальный prebuilt venv на хосте Jenkins (Windows service).
        SHARED_VENV = 'C:\\Mlops_2\\venv'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup & Cache') {
            steps {
                bat '''
                    if not exist "%SHARED_VENV%\\Scripts\\activate.bat" (
                        echo Shared venv not found: %SHARED_VENV%
                        exit /b 1
                    )
                    call "%SHARED_VENV%\\Scripts\\activate.bat"
                    python --version
                    pip --version
                '''
            }
        }

        stage('Lint & Test') {
            parallel {
                stage('Lint') {
                    steps {
                        bat '''
                            call "%SHARED_VENV%\\Scripts\\activate.bat"
                            black src tests
                            mypy src tests --follow-untyped-imports
                        '''
                    }
                }
                stage('Unit Tests') {
                    steps {
                        withCredentials([file(credentialsId: "${GDRIVE_CREDENTIALS_ID}", variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                            bat '''
                                call "%SHARED_VENV%\\Scripts\\activate.bat"
                                copy /Y "%GOOGLE_APPLICATION_CREDENTIALS%" "service_account.json"
                                dvc pull --jobs 4
                                pytest --maxfail=1 --disable-warnings -q tests/test_data_quality.py
                            '''
                        }
                    }
                }
            }
        }

        stage('Train Model') {
            steps {
                withCredentials([file(credentialsId: "${GDRIVE_CREDENTIALS_ID}", variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    bat '''
                        call "%SHARED_VENV%\\Scripts\\activate.bat"
                        copy /Y "%GOOGLE_APPLICATION_CREDENTIALS%" "service_account.json"
                        dvc pull --jobs 4
                        python -m src.services.model_pipeline.pipeline
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'models/**', fingerprint: true
                }
            }
        }

        stage('Integration Tests') {
            steps {
                withCredentials([file(credentialsId: "${GDRIVE_CREDENTIALS_ID}", variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    bat '''
                        call "%SHARED_VENV%\\Scripts\\activate.bat"
                        copy /Y "%GOOGLE_APPLICATION_CREDENTIALS%" "service_account.json"
                        dvc pull --jobs 4
                        pytest --maxfail=1 --disable-warnings -q tests/test_endpoints.py
                    '''
                }
            }
        }

        stage('Build & Push Docker') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    bat '''
                        docker --version
                        docker login -u "%DOCKER_USERNAME%" -p "%DOCKER_PASSWORD%"
                        docker build -t %DOCKER_IMAGE_NAME%:%DOCKER_IMAGE_TAG% .
                        docker push %DOCKER_IMAGE_NAME%:%DOCKER_IMAGE_TAG%
                        docker tag %DOCKER_IMAGE_NAME%:%DOCKER_IMAGE_TAG% %DOCKER_IMAGE_NAME%:latest
                        docker push %DOCKER_IMAGE_NAME%:latest
                    '''
                }
            }
        }
    }
    post {
        success {
            echo '✅ Сборка успешно завершена'
        }
        failure {
            echo '❌ Сборка упала'
        }
        always {
            archiveArtifacts artifacts: 'data/processed/**/*', fingerprint: true
        }
    }
}