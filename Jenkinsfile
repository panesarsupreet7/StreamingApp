pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '444068947659'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        FRONTEND_IMAGE = "${ECR_REGISTRY}/streamingapp-frontend"
        AUTH_IMAGE = "${ECR_REGISTRY}/streamingapp-auth"
        STREAMING_IMAGE = "${ECR_REGISTRY}/streamingapp-streaming"
        ADMIN_IMAGE = "${ECR_REGISTRY}/streamingapp-admin"
        CHAT_IMAGE = "${ECR_REGISTRY}/streamingapp-chat"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    def gitSha = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = gitSha

                    sh """
                        docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ./frontend

                        docker build -t ${AUTH_IMAGE}:${IMAGE_TAG} ./backend/authService

                        docker build -t ${STREAMING_IMAGE}:${IMAGE_TAG} \
                            -f ./backend/streamingService/Dockerfile ./backend

                        docker build -t ${ADMIN_IMAGE}:${IMAGE_TAG} \
                            -f ./backend/adminService/Dockerfile ./backend

                        docker build -t ${CHAT_IMAGE}:${IMAGE_TAG} \
                            -f ./backend/chatService/Dockerfile ./backend
                    """
                }
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} |
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    docker push ${AUTH_IMAGE}:${IMAGE_TAG}
                    docker push ${STREAMING_IMAGE}:${IMAGE_TAG}
                    docker push ${ADMIN_IMAGE}:${IMAGE_TAG}
                    docker push ${CHAT_IMAGE}:${IMAGE_TAG}
                """
            }
        }
    }

    post {
        success {
            echo 'All Docker images were successfully built and pushed to ECR.'
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }
    }
}