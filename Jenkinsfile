pipeline { 
    agent any

    triggers {
        githubPush()
    }
 
    environment { 
        AWS_REGION = 'ap-south-1' 
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

                docker build -t ${STREAMING_IMAGE}:${IMAGE_TAG} -f ./backend/streamingService/Dockerfile ./backend

                docker build -t ${ADMIN_IMAGE}:${IMAGE_TAG} -f ./backend/adminService/Dockerfile ./backend

                docker build -t ${CHAT_IMAGE}:${IMAGE_TAG} -f ./backend/chatService/Dockerfile ./backend
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
 
 stage('Deploy to EC2') {
    steps {
        sh """
            echo "Logging in to ECR..."

            aws ecr get-login-password --region ${AWS_REGION} |
            docker login --username AWS --password-stdin ${ECR_REGISTRY}

            echo "Pulling images..."

            docker pull ${FRONTEND_IMAGE}:${IMAGE_TAG}
            docker pull ${AUTH_IMAGE}:${IMAGE_TAG}
            docker pull ${STREAMING_IMAGE}:${IMAGE_TAG}
            docker pull ${ADMIN_IMAGE}:${IMAGE_TAG}
            docker pull ${CHAT_IMAGE}:${IMAGE_TAG}

            echo "Creating Docker network if needed..."

            docker network create streaming-network 2>/dev/null || true

            echo "Starting MongoDB..."

            docker rm -f streaming-mongo 2>/dev/null || true

            docker run -d \
                --name streaming-mongo \
                --network streaming-network \
                --restart unless-stopped \
                -p 27017:27017 \
                mongo:6

            echo "Stopping old application containers..."

            docker rm -f streaming-frontend streaming-auth streaming-streaming streaming-admin streaming-chat 2>/dev/null || true

            echo "Starting Auth service..."

            docker run -d \
                --name streaming-auth \
                --network streaming-network \
                --restart unless-stopped \
                -p 3001:3001 \
                -e PORT=3001 \
                -e MONGO_URI=mongodb://streaming-mongo:27017/streamingapp \
                -e JWT_SECRET=changeme \
                ${AUTH_IMAGE}:${IMAGE_TAG}

            echo "Starting Streaming service..."

            docker run -d \
                --name streaming-streaming \
                --network streaming-network \
                --restart unless-stopped \
                -p 3002:3002 \
                -e PORT=3002 \
                -e MONGO_URI=mongodb://streaming-mongo:27017/streamingapp \
                -e JWT_SECRET=changeme \
                ${STREAMING_IMAGE}:${IMAGE_TAG}

            echo "Starting Admin service..."

            docker run -d \
                --name streaming-admin \
                --network streaming-network \
                --restart unless-stopped \
                -p 3003:3003 \
                -e PORT=3003 \
                -e MONGO_URI=mongodb://streaming-mongo:27017/streamingapp \
                -e JWT_SECRET=changeme \
                ${ADMIN_IMAGE}:${IMAGE_TAG}

            echo "Starting Chat service..."

            docker run -d \
                --name streaming-chat \
                --network streaming-network \
                --restart unless-stopped \
                -p 3004:3004 \
                -e PORT=3004 \
                -e MONGO_URI=mongodb://streaming-mongo:27017/streamingapp \
                -e JWT_SECRET=changeme \
                ${CHAT_IMAGE}:${IMAGE_TAG}

            echo "Starting Frontend..."

            docker run -d \
                --name streaming-frontend \
                --network streaming-network \
                --restart unless-stopped \
                -p 3000:80 \
                ${FRONTEND_IMAGE}:${IMAGE_TAG}

            echo "Checking running containers..."

            docker ps

            echo "Deployment completed."
        """
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