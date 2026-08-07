pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_REPOSITORY = 'ai-agent-hub'
        AWS_ACCOUNT_ID = sh(
            script: 'aws sts get-caller-identity --query Account --output text',
            returnStdout: true
        ).trim()
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_IMAGE = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Maven Build & Test') {
            steps {
                echo 'Building and testing Spring Boot application...'
                sh 'mvn -B clean package'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ai-agent-hub:${BUILD_NUMBER}"
                sh "docker build -t ai-agent-hub:${BUILD_NUMBER} ."
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                echo 'Logging in to Amazon ECR...'
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Tag Docker Image') {
            steps {
                echo "Tagging image as ${ECR_IMAGE}"
                sh "docker tag ai-agent-hub:${BUILD_NUMBER} ${ECR_IMAGE}"
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                echo "Pushing ${ECR_IMAGE} to Amazon ECR..."
                sh "docker push ${ECR_IMAGE}"
            }
        }

        stage('Stop Existing Container') {
            steps {
                echo 'Stopping existing container...'
                sh 'docker stop ai-agent-hub || true'
            }
        }

        stage('Remove Existing Container') {
            steps {
                echo 'Removing existing container...'
                sh 'docker rm ai-agent-hub || true'
            }
        }

        stage('Run New Container') {
            steps {
                echo "Starting container using image ${ECR_IMAGE}..."

                sh "docker run -d --name ai-agent-hub -p 8081:8081 ${ECR_IMAGE}"
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Verifying application...'
                sh 'sleep 10'
                sh 'curl -f http://localhost:8081/ || exit 1'
                echo 'Application is running successfully!'
            }
        }

        stage('Docker Cleanup') {
            steps {
                echo 'Cleaning up unused Docker images...'
                sh 'docker image prune -f'
            }
        }
    }
}
