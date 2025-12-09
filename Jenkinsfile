pipeline {

    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: maven
      image: maven:3.9.6-eclipse-temurin-17
      command: ['cat']
      tty: true

    - name: docker
      image: docker:26.1-dind
      securityContext:
        privileged: true
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""

    - name: docker-cli
      image: docker:26.1-cli
      command: ['cat']
      tty: true

    - name: kubectl
      image: bitnami/kubectl:latest
      command: ['cat']
      tty: true

  volumes:
    - name: dockersock
      emptyDir: {}
"""
        }
    }

    environment {
        APP_NAME    = "complete-pipeline"
        RELEASE     = "1.0.0"
        DOCKER_USER = "shabaz7323"
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
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=${APP_NAME} \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.login=$SONAR_AUTH_TOKEN
                        """
                    }
                }
            }
        }

        stage('SonarQube Quality Gate (Webhook Async)') {
            steps {
                script {
                    echo "⏳ Waiting for SonarQube webhook callback (true async)..."

                    timeout(time: 10, unit: "MINUTES") {

                        // The webhookSecretId MUST match the credential you added in Jenkins
                        def qg = waitForQualityGate(
                            webhookSecretId: 'sonar-webhook-secret'
                        )

                        if (qg.status != "OK") {
                            error "❌ Quality Gate FAILED: ${qg.status}"
                        }

                        echo "✅ Quality Gate PASSED: ${qg.status}"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker-cli') {
                    sh """
                        docker -H tcp://localhost:2375 build -t ${IMAGE_NAME}:${RELEASE} .
                    """
                }
            }
        }

        stage('Docker Login') {
            steps {
                container('docker-cli') {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'USERNAME',
                        passwordVariable: 'PASSWORD'
                    )]) {
                        sh 'echo $PASSWORD | docker -H tcp://localhost:2375 login -u $USERNAME --password-stdin'
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker-cli') {
                    sh "docker -H tcp://localhost:2375 push ${IMAGE_NAME}:${RELEASE}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh """
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✨ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed — check logs."
        }
    }
}
