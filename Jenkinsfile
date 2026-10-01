pipeline {
    agent any

    environment {
        PATH = "/home/malek/.nvm/versions/node/v22.23.3/bin:${env.PATH}"
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Backend - Build') {
            steps {
                dir('backend') {
                    sh './mvnw clean package -DskipTests'
                }
            }
        }

        stage('Frontend - Install') {
            steps {
                dir('frontend') {
                    sh '''
                        node --version
                        npm --version
                        npm ci
                    '''
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
                    echo "===== Docker Build ====="

                    docker build -t backend-app:latest ./backend

                    docker tag backend-app:latest \
                        localhost:5000/backend-app:latest
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
                        echo "===== Docker Registry Login ====="

                        echo "$REGISTRY_PASSWORD" | docker login localhost:5000 \
                            --username "$REGISTRY_USER" \
                            --password-stdin

                        echo "===== Docker Push ====="

                        docker push localhost:5000/backend-app:latest

                        docker logout localhost:5000
                    '''
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh '''
                    echo "===== Deploy MySQL ====="

                    if docker ps -a --format '{{.Names}}' | grep -q '^mysql$'; then

                        echo "MySQL container already exists."

                        if [ "$(docker inspect -f '{{.State.Running}}' mysql)" = "false" ]; then
                            echo "Starting MySQL..."
                            docker start mysql
                        else
                            echo "MySQL is already running."
                        fi

                    else
                        echo "ERROR: MySQL container does not exist."
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
                        echo "===== Stop old backend containers ====="

                        docker stop backend-app 2>/dev/null || true
                        docker rm backend-app 2>/dev/null || true

                        docker stop appgestion-backend 2>/dev/null || true
                        docker rm appgestion-backend 2>/dev/null || true

                        echo "===== Docker Registry Login ====="

                        echo "$REGISTRY_PASSWORD" | docker login localhost:5000 \
                            --username "$REGISTRY_USER" \
                            --password-stdin

                        echo "===== Pull latest backend image ====="

                        docker pull localhost:5000/backend-app:latest

                        echo "===== Start backend-app ====="

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

