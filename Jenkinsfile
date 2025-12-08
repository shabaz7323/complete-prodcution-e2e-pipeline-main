pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-docker-agent
spec:
  serviceAccountName: jenkins

  containers:
  - name: docker
    image: docker:26-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
    command:
    - dockerd-entrypoint.sh
    args:
    - --host=tcp://0.0.0.0:2375
    - --tls=false
    volumeMounts:
    - name: docker-storage
      mountPath: /var/lib/docker

  - name: docker-cli
    image: docker:26-cli
    env:
    - name: DOCKER_HOST
      value: tcp://localhost:2375
    - name: DOCKER_TLS_CERTDIR
      value: ""
    command:
    - cat
    tty: true

  - name: maven
    image: maven:3.9.8-eclipse-temurin-17
    command:
    - cat
    tty: true

  - name: kubectl
    image: alpine/k8s:1.30.0
    command:
    - sh
    - -c
    - cat
    tty: true

  volumes:
  - name: docker-storage
    emptyDir: {}
"""
        }
    }

    environment {
        APP_NAME    = "complete-prodcution-e2e-pipeline"
        RELEASE     = "1.0.0"
        DOCKER_USER = "shabaz7323"
        IMAGE_NAME  = "${DOCKER_USER}/${APP_NAME}"

        SONAR_AUTH_TOKEN = credentials('sonar-token')
    }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/shabaz7323/complete-prodcution-e2e-pipeline-main.git'
            }
        }

       

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            mvn clean verify sonar:sonar \
                                -Dsonar.projectKey=complete-pipeline \
                                -Dsonar.projectName=complete-pipeline \
                                -Dsonar.host.url=http://sonar-sonarqube:9000 \
                                -Dsonar.login=$SONAR_AUTH_TOKEN
                        """
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                script {
                    timeout(time: 3, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker-cli') {
                    sh """
                        docker build -t ${IMAGE_NAME}:${RELEASE} .
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
                        sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker-cli') {
                    sh "docker push ${IMAGE_NAME}:${RELEASE}"
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
            echo "✨ Pipeline completed successfully with SonarQube + Docker + Kubernetes!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
