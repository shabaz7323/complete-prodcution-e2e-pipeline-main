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

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME    = "complete-prodcution-e2e-pipeline"
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

       
        stage('Build Docker Image') {
            steps {
                container('docker-cli') {
                    sh """
                        echo "--- DOCKER VERSION ---"
                        docker version

                        echo "--- BUILDING IMAGE ---"
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
                        echo "--- APPLYING DEPLOYMENT ---"
                        kubectl apply -f k8s/deployment.yaml
                        
                        echo "--- APPLYING SERVICE ---"
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
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
