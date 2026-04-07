pipeline {
    agent none

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
    }

    stages {

        stage('Checkout') {
            agent any
            steps {
                checkout scm
            }
        }

        stage('Setup & Cache') {
            agent {
                docker {
                    image 'python:3.12-slim'
                    args  '--network host \
                           -v $HOME/.cache/pip:/root/.cache/pip'
                }
            }
            steps {
                sh '''
                    python -m venv venv
                    . venv/bin/activate
                    pip install 'dvc[gdrive]'
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Lint & Test') {
            parallel {
                stage('Lint') {
                    agent {
                        docker { image 'python:3.12-slim'; args '--network host' }
                    }
                    steps {
                        sh '''
                            set -e
                            . venv/bin/activate
                            black src tests
                            mypy src tests --follow-untyped-imports
                        '''
                    }
                }
                stage('Unit Tests') {
                    agent {
                        docker { image 'python:3.12-slim'; args '--network host' }
                    }
                    steps {
                        withCredentials([file(credentialsId: "${GDRIVE_CREDENTIALS_ID}", variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                            sh '''
                                set -e
                                . venv/bin/activate
                                # Put the secret JSON into repo root so DVC can use the relative path.
                                cp "$GOOGLE_APPLICATION_CREDENTIALS" ./service_account.json
                                dvc pull --jobs 4
                                pytest --maxfail=1 --disable-warnings -q tests/test_data_quality.py
                            '''
                        }
                    }
                }
            }
        }

        stage('Train Model') {
            agent {
                docker { image 'python:3.12-slim'; args '--network host' }
            }
            steps {
                withCredentials([file(credentialsId: "${GDRIVE_CREDENTIALS_ID}", variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        set -e
                        . venv/bin/activate
                        # Put the secret JSON into repo root so DVC can use the relative path.
                        cp "$GOOGLE_APPLICATION_CREDENTIALS" ./service_account.json
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
            agent {
                docker { image 'python:3.12-slim'; args '--network host' }
            }
            steps {
                withCredentials([file(credentialsId: "${GDRIVE_CREDENTIALS_ID}", variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        set -e
                        . venv/bin/activate
                        # Put the secret JSON into repo root so DVC can use the relative path.
                        cp "$GOOGLE_APPLICATION_CREDENTIALS" ./service_account.json
                        dvc pull --jobs 4
                        pytest --maxfail=1 --disable-warnings -q tests/test_endpoints.py
                    '''
                }
            }
        }

        stage('Build & Push Docker') {
            agent {
                docker {
                    image 'docker:23.0-dind'
                    args  '--privileged --network host'
                }
            }
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", "${DOCKER_CREDENTIALS_ID}") {
                        def img = docker.build("${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}")
                        img.push()
                        img.push('latest')
                    }
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