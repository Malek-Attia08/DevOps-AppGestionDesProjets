
pipeline {
    agent any

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Backend - Test & Build') {
            steps {
                dir('backend') {
                    sh './mvnw clean test package'
                }
            }
        }

        stage('Frontend - Install') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                }
            }
        }

        stage('Frontend - Test') {
            steps {
                dir('frontend') {
                    sh 'npm test -- --watch=false'
                }
            }
        }

        stage('Frontend - Build') {
            steps {
                dir('frontend') {
                    sh 'npm run build'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t backend-app:latest ./backend
                    docker tag backend-app:latest localhost:5000/backend-app:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-registry-credentials',
                        usernameVariable: 'REGISTRY_USER',
                        passwordVariable: 'REGISTRY_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$REGISTRY_PASSWORD" | docker login localhost:5000 \
                            --username "$REGISTRY_USER" \
                            --password-stdin

                        docker push localhost:5000/backend-app:latest

                        docker logout localhost:5000
                    '''
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh '''
                    if docker ps -a --format '{{.Names}}' | grep -q '^mysql$'; then

                        echo "MySQL container exists."

                        if [ "$(docker inspect -f '{{.State.Running}}' mysql)" = "false" ]; then
                            echo "Starting MySQL..."
                            docker start mysql
                        else
                            echo "MySQL is already running."
                        fi

                    else
                        echo "MySQL container does not exist."
                        exit 1
                    fi
                '''
            }
        }

        stage('Deploy Backend') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-registry-credentials',
                        usernameVariable: 'REGISTRY_USER',
                        passwordVariable: 'REGISTRY_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "Stopping old backend-app container..."
                        docker stop backend-app 2>/dev/null || true

                        echo "Removing old backend-app container..."
                        docker rm backend-app 2>/dev/null || true

                        echo "Logging into Docker registry..."
                        echo "$REGISTRY_PASSWORD" | docker login localhost:5000 \
                            --username "$REGISTRY_USER" \
                            --password-stdin

                        echo "Pulling latest backend image..."
                        docker pull localhost:5000/backend-app:latest

                        echo "Starting new backend-app container..."
                        docker run -d \
                            --name backend-app \
                            -p 8090:8080 \
                            --link mysql:mysql \
                            localhost:5000/backend-app:latest

                        docker logout localhost:5000
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "===== Docker Containers ====="
                    docker ps

                    echo "===== Backend Logs ====="
                    docker logs backend-app
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'CI/CD pipeline failed.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
        }
    }
}
