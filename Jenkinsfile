pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = sh(
            script: 'aws sts get-caller-identity --query Account --output text',
            returnStdout: true
        ).trim()

        ECR_REPOSITORY = 'ai-agent-hub'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_IMAGE = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"

        S3_BUCKET = "ai-agent-hub-artifacts-${AWS_ACCOUNT_ID}"
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
                echo 'Waiting for Kubernetes rollout...'

                sh '''
                    kubectl rollout status deployment/ai-agent-hub \
                    --timeout=180s
                '''
            }
        }

        stage('Verify Kubernetes Deployment') {
            steps {
                echo 'Verifying Kubernetes deployment...'

                sh '''
                    kubectl get deployment ai-agent-hub
                    kubectl get pods -l app=ai-agent-hub
                    kubectl get service ai-agent-hub-service
                '''
            }
        }

        stage('Upload Artifacts to S3') {
            steps {
                echo "Uploading build ${BUILD_NUMBER} artifacts to S3..."

                sh '''
                    mkdir -p artifact-metadata

                    echo "Build Number: ${BUILD_NUMBER}" \
                        > artifact-metadata/build-info.txt

                    echo "Git Commit: ${GIT_COMMIT:-unknown}" \
                        >> artifact-metadata/build-info.txt

                    echo "ECR Image: ${ECR_IMAGE}" \
                        >> artifact-metadata/build-info.txt

                    echo "Build Date: $(date -u)" \
                        >> artifact-metadata/build-info.txt

                    aws s3 cp target/*.jar \
                        s3://${S3_BUCKET}/builds/${BUILD_NUMBER}/

                    aws s3 cp target/surefire-reports/ \
                        s3://${S3_BUCKET}/test-reports/${BUILD_NUMBER}/ \
                        --recursive || true

                    aws s3 cp artifact-metadata/build-info.txt \
                        s3://${S3_BUCKET}/metadata/${BUILD_NUMBER}/build-info.txt
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
}
