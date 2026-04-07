pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDS = credentials('dockerhub-creds')
        DOCKER_HUB_USER  = "${DOCKER_HUB_CREDS_USR}"
        EC2_HOST         = credentials('ec2-host')          // e.g. ubuntu@<EC2-IP>
        SSH_KEY          = credentials('ec2-ssh-key')       // SSH private key
        SONAR_TOKEN      = credentials('sonar-token')
        SONAR_HOST_URL   = 'http://localhost:9000'          // update if SonarQube is on separate host
        IMAGE_TAG        = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('project') {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            sonar-scanner \
                              -Dsonar.projectKey=microservices-notes-app \
                              -Dsonar.sources=. \
                              -Dsonar.exclusions=**/node_modules/** \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.token=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                dir('project') {
                    sh """
                        docker build -t ${DOCKER_HUB_USER}/api-gateway:${IMAGE_TAG}    ./api-gateway
                        docker build -t ${DOCKER_HUB_USER}/user-service:${IMAGE_TAG}   ./user-service
                        docker build -t ${DOCKER_HUB_USER}/notes-service:${IMAGE_TAG}  ./notes-service
                        docker build -t ${DOCKER_HUB_USER}/frontend:${IMAGE_TAG}       ./frontend
                    """
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh """
                    trivy image --exit-code 1 --severity HIGH,CRITICAL \
                        --no-progress ${DOCKER_HUB_USER}/api-gateway:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity HIGH,CRITICAL \
                        --no-progress ${DOCKER_HUB_USER}/user-service:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity HIGH,CRITICAL \
                        --no-progress ${DOCKER_HUB_USER}/notes-service:${IMAGE_TAG}

                    trivy image --exit-code 1 --severity HIGH,CRITICAL \
                        --no-progress ${DOCKER_HUB_USER}/frontend:${IMAGE_TAG}
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh "echo ${DOCKER_HUB_CREDS_PSW} | docker login -u ${DOCKER_HUB_USER} --password-stdin"
                sh """
                    docker push ${DOCKER_HUB_USER}/api-gateway:${IMAGE_TAG}
                    docker push ${DOCKER_HUB_USER}/user-service:${IMAGE_TAG}
                    docker push ${DOCKER_HUB_USER}/notes-service:${IMAGE_TAG}
                    docker push ${DOCKER_HUB_USER}/frontend:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_HOST} '
                            cd /home/ubuntu/microservices_project/project &&
                            export IMAGE_TAG=${IMAGE_TAG} &&
                            export DOCKER_HUB_USER=${DOCKER_HUB_USER} &&
                            docker compose pull &&
                            docker compose up -d --remove-orphans
                        '
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker logout"
            cleanWs()
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
        success {
            echo "Deployment successful. App running on EC2."
        }
    }
}
