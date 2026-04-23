pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        NEXUS_HOST = "nexus"
        NEXUS_PORT = "8081"
        NEXUS_URL = "http://${NEXUS_HOST}:${NEXUS_PORT}"
        NEXUS_REPO = "docker-dev"
        
        IMAGE_NAME = "webapp"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        DOCKER_REGISTRY = "${NEXUS_HOST}:8082"
        IMAGE_FULL_NAME = "${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        IMAGE_LATEST = "${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
        
        NEXUS_CREDENTIALS_ID = "nexus-credentials"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Obteniendo código..."
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Dependency Analysis') {
            steps {
                echo "Analizando dependencias con npm audit..."
                sh '''
                    docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm audit || true
                '''
            }
        }

        stage('Run tests') {
            steps {
                sh "docker run ${IMAGE_NAME}:${IMAGE_TAG} npm test"
            }
        }

        stage('Security Scanning') {
            steps {
                echo "Escaneando con Trivy..."
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        ${IMAGE_NAME}:${IMAGE_TAG} || EXIT_CODE=$?
                    
                    if [ $EXIT_CODE -eq 1 ]; then
                        echo "ERROR: Vulnerabilidades críticas detectadas"
                        exit 1
                    fi
                '''
            }
        }

        stage('Tag Docker Image') {
            steps {
                echo "Tagging Docker image..."
                sh '''
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_FULL_NAME}
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_LATEST}
                '''
            }
        }

        stage('Deploy Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${NEXUS_CREDENTIALS_ID}", 
                                                   usernameVariable: 'NEXUS_USER', 
                                                   passwordVariable: 'NEXUS_PASSWORD')]) {
                    sh '''
                        echo "${NEXUS_PASSWORD}" | docker login -u "${NEXUS_USER}" --password-stdin ${DOCKER_REGISTRY}
                        docker push ${IMAGE_FULL_NAME}
                        docker push ${IMAGE_LATEST}
                        docker logout ${DOCKER_REGISTRY}
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up local Docker images..."
            sh '''
            docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
            docker rmi ${IMAGE_FULL_NAME} || true
            docker rmi ${IMAGE_LATEST} || true
            '''
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check the logs for details."
        }
    }
}