pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME    = "complete-prodcution-e2e-pipeline"
        RELEASE     = "1.0.0"
        DOCKER_USER = "dmancloud"
        IMAGE_NAME  = "${DOCKER_USER}/${APP_NAME}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/shabaz7323/complete-prodcution-e2e-pipeline-main.git'
            }
        }

        stage('Build Maven Project') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker build -t ${IMAGE_NAME}:${RELEASE} .
                    """
                }
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'PASSWORD'
                )]) {
                    sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}:${RELEASE}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                """
            }
        }
    }

    post {
        success {
            echo "✨ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
