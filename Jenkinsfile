pipeline {
    agent any

    stages {

        stage('Checkout Code') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                echo 'Building Spring Boot application...'
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t ai-agent-hub:latest .'
            }
        }

        stage('Stop Existing Container') {
            steps {
                echo 'Stopping existing container...'
                sh 'docker stop ai-agent-hub || true'
            }
        }

    }
}
