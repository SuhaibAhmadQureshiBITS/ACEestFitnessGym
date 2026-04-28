pipeline {
    agent any

    environment {
        IMAGE_NAME  = "aceest-fitness"
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        LOCAL_BIN   = "/root/.local/bin"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Source code checked out from GitHub"
            }
        }

        stage('Build, Lint & Test') {
            agent {
                docker {
                    image 'python:3.11'
                    args '-u root'   // needed for pip install
                }
            }
            steps {
                sh 'pip install --break-system-packages -r requirements.txt'
                sh '${LOCAL_BIN}/flake8 app.py test_app.py --max-line-length=100 || true'
                sh 'python -m pytest test_app.py -v --tb=short'
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                echo "Docker image built: ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Quality Gate') {
            steps {
                sh """
                    docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} \
                        python3 -m pytest test_app.py -v --tb=short
                """
                echo "Containerized tests passed — Quality Gate cleared"
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS — Build #${env.BUILD_NUMBER} passed all stages"
        }
        failure {
            echo "Pipeline FAILED — Review logs for Build #${env.BUILD_NUMBER}"
        }
        always {
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        }
    }
}
