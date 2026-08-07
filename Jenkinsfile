pipeline {
    agent any

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
                echo "Starting container using image ai-agent-hub:${BUILD_NUMBER}..."
                sh "docker run -d --name ai-agent-hub -p 8081:8081 ai-agent-hub:${BUILD_NUMBER}"
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
    }
}
