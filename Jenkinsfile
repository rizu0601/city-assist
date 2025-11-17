pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        SSH_HOST    = "ubuntu@172.31.16.10"
        DEPLOY_PATH = "/home/ubuntu/Hackathon"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Backend JAR') {
            steps {
                dir('backend') {
                    sh "mvn clean package -DskipTests"
                }
            }
        }

        stage('Build Docker Images Locally') {
            steps {
                sh """
                    docker build -t hack-backend:latest backend/
                    docker build -t hack-frontend:latest frontend/
                    docker build -t hack-python:latest python/
                """
            }
        }

        stage('Copy Files to EC2') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'ansible-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {

                        sh """
                            echo "📤 Copying docker-compose.yml to server..."
                            scp -o StrictHostKeyChecking=no -i $SSH_KEY docker-compose.yml $SSH_HOST:$DEPLOY_PATH/
                        """
                    }
                }
            }
        }

        stage('Deploy on Same Server') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'ansible-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {

                        sh """
                            ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_HOST '
                                cd $DEPLOY_PATH

                                echo "🔻 Stopping old containers"
                                docker-compose down || true

                                echo "🚀 Starting new containers"
                                docker-compose up -d --build --force-recreate

                                echo "🧹 Cleaning unused images"
                                docker system prune -a -f
                            '
                        """
                    }
                }
            }
        }
    }

    post {
        success { echo "✅ Deployment successful!" }
        failure { echo "❌ Deployment failed!" }
    }
}
