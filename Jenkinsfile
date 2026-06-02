pipeline {
    agent any
    
    environment {
        APP_PORT = '5556'
        IMAGE_NAME = 'hello-spencer-app'
        CONTAINER_NAME = 'hello-spencer-container'
    }
    
    stages {
        stage('Source') {
            steps {
                cleanWs()
                checkout scm
            }
        }
        
        stage('Test') {
            agent {
                docker { 
                    image 'python:3.11-slim-buster'
                    args '-u root'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # Install dependencies and test tools
                    python -m pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install pytest
                    
                    # Ensure count.txt exists since hello.py needs it
                    if [ ! -f count.txt ]; then
                        echo "0" > count.txt
                    fi
                    chmod 666 count.txt
                    
                    # Run the unit tests
                    python -m pytest tests/test_hello.py -v
                '''
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    // Build the docker image
                    docker.build("${IMAGE_NAME}:${env.BUILD_ID}")
                }
            }
        }
        
        stage('Deployment') {
            steps {
                sh '''
                    # Stop and remove any existing container
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    
                    # Ensure count.txt exists before deployment
                    if [ ! -f count.txt ]; then
                        echo "0" > count.txt
                    fi
                    chmod 666 count.txt
                    
                    # Run the new container locally
                    docker run -d --name ${CONTAINER_NAME} -p ${APP_PORT}:${APP_PORT} ${IMAGE_NAME}:${BUILD_ID}
                '''
            }
        }
        
        stage('Integration Test') {
            steps {
                sh '''
                    # Wait for application to start
                    sleep 5
                    
                    # Get the container's IP address dynamically
                    CONTAINER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${CONTAINER_NAME})
                    
                    # Test if the API is responding using the container IP
                    curl http://${CONTAINER_IP}:${APP_PORT}/api/hello
                '''
            }
        }
    }
}