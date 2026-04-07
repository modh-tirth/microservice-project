pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://localhost:9000'
        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    dir('project') {
                        withSonarQubeEnv('SonarQube') {
                            sh '''
                                sonar-scanner \
                                  -Dsonar.projectKey=microservices-notes-app \
                                  -Dsonar.sources=. \
                                  -Dsonar.exclusions=**/node_modules/** \
                                  -Dsonar.host.url=$SONAR_HOST_URL \
                                  -Dsonar.token=$SONAR_TOKEN
                            '''
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                dir('project') {
                    sh """
                        docker build -t api-gateway:${IMAGE_TAG}   ./api-gateway
                        docker build -t user-service:${IMAGE_TAG}  ./user-service
                        docker build -t notes-service:${IMAGE_TAG} ./notes-service
                        docker build -t frontend:${IMAGE_TAG}      ./frontend
                    """
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh """
                    trivy image --exit-code 1 --severity HIGH,CRITICAL --no-progress api-gateway:${IMAGE_TAG}
                    trivy image --exit-code 1 --severity HIGH,CRITICAL --no-progress user-service:${IMAGE_TAG}
                    trivy image --exit-code 1 --severity HIGH,CRITICAL --no-progress notes-service:${IMAGE_TAG}
                    trivy image --exit-code 1 --severity HIGH,CRITICAL --no-progress frontend:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy') {
            steps {
                dir('project') {
                    sh "IMAGE_TAG=${IMAGE_TAG} docker-compose up -d --remove-orphans"
                }
            }
        }
    }

    post {
        always {
            node('') {
                cleanWs()
            }
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
        success {
            echo "Deployment successful. App running on EC2."
        }
    }
}
