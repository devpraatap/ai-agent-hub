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

        stage('Deploy to Kubernetes') {
            steps {
                echo "Deploying ${ECR_IMAGE} to Kubernetes..."

                sh """
                    kubectl set image deployment/ai-agent-hub \
                    ai-agent-hub=${ECR_IMAGE}
                """
            }
        }

        stage('Kubernetes Rollout Status') {
            steps {
                echo 'Waiting for Kubernetes rollout to complete...'

                sh '''
                    kubectl rollout status deployment/ai-agent-hub --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Verifying Kubernetes deployment...'

                sh '''
                    echo "===== Deployment ====="
                    kubectl get deployment ai-agent-hub

                    echo "===== Pods ====="
                    kubectl get pods -l app=ai-agent-hub

                    echo "===== Current Image ====="
                    kubectl get deployment ai-agent-hub \
                      -o jsonpath='{.spec.template.spec.containers[0].image}'
                    echo

                    echo "===== Available Replicas ====="
                    kubectl get deployment ai-agent-hub \
                      -o jsonpath='{.status.availableReplicas}'
                    echo
                '''
            }
        }

        stage('Docker Cleanup') {
            steps {
                echo 'Cleaning up unused Docker images...'

                sh 'docker image prune -f'
            }
        }
    }

    post {

        failure {
            echo 'Deployment failed. Collecting Kubernetes diagnostics...'

            sh '''
                echo "===== Deployment Status ====="
                kubectl get deployment ai-agent-hub || true

                echo "===== Pods ====="
                kubectl get pods -l app=ai-agent-hub -o wide || true

                echo "===== ReplicaSets ====="
                kubectl get replicasets -l app=ai-agent-hub || true

                echo "===== Recent Events ====="
                kubectl get events --sort-by=.lastTimestamp | tail -30 || true
            '''
        }

        success {
            echo '========================================'
            echo 'CI/CD DEPLOYMENT SUCCESSFUL'
            echo '========================================'
        }
    }
}
