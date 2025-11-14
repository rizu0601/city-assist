pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        REGISTRY       = 'docker.io/swapnilneo'
        BACKEND_IMAGE  = 'hack-backend'
        FRONTEND_IMAGE = 'hack-frontend'
        PYTHON_IMAGE   = 'hack-python'
        TAG            = "${env.BUILD_NUMBER}"
        SSH_HOST       = 'ubuntu@172.31.16.10'
        DEPLOY_PATH    = '/home/ubuntu/Hackathon'
        TAG_FILE       = '/home/ubuntu/Hackathon/.last_successful_tag'
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

        stage('Build & Push Docker Images') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-hub-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                            docker build -t $REGISTRY/$BACKEND_IMAGE:$TAG backend/
                            docker build -t $REGISTRY/$FRONTEND_IMAGE:$TAG frontend/
                            docker build -t $REGISTRY/$PYTHON_IMAGE:$TAG python/

                            docker push $REGISTRY/$BACKEND_IMAGE:$TAG
                            docker push $REGISTRY/$FRONTEND_IMAGE:$TAG
                            docker push $REGISTRY/$PYTHON_IMAGE:$TAG
                        '''
                    }
                }
            }
        }

        stage('Copy docker-compose.yml If Changed') {
            steps {
                script {
                    def changed = sh(
                        script: "git diff --name-only HEAD~1 HEAD | grep 'docker-compose.yml' || true",
                        returnStdout: true
                    ).trim()

                    if (changed) {
                        echo "docker-compose.yml changed → Copying to EC2"

                        withCredentials([sshUserPrivateKey(
                            credentialsId: 'ansible-ssh-key',
                            keyFileVariable: 'SSH_KEY'
                        )]) {

                            sh '''
                                scp -o StrictHostKeyChecking=no -i $SSH_KEY docker-compose.yml $SSH_HOST:$DEPLOY_PATH/
                            '''
                        }
                    } else {
                        echo "docker-compose.yml not changed → Skipping copy"
                    }
                }
            }
        }

        stage('Deploy to EC2 with Rollback') {
            steps {
                script {
                    withCredentials([
                        sshUserPrivateKey(
                            credentialsId: 'ansible-ssh-key',
                            keyFileVariable: 'SSH_KEY'
                        )
                    ]) {

                        try {

                            sh '''
                                ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_HOST '
                                    echo "📌 Saving previous TAG..."
                                    PREV_TAG="unknown"
                                    if [ -f $TAG_FILE ]; then
                                        PREV_TAG=$(cat $TAG_FILE)
                                    fi
                                    echo "Previous Tag: $PREV_TAG"

                                    echo "🔻 Stopping old containers..."
                                    docker-compose down || true

                                    echo "📥 Pulling new images with TAG: '"$TAG"'..."
                                    export TAG='"$TAG"'
                                    docker-compose pull

                                    echo "🚀 Starting new containers..."
                                    docker-compose up -d --force-recreate

                                    echo "💾 Saving new TAG to file..."
                                    echo '"$TAG"' > $TAG_FILE

                                    echo "🧹 Cleaning unused Docker data..."
                                    docker system prune -a -f --volumes
                                '
                            '''

                        } catch (Exception e) {

                            echo "❌ Deployment FAILED — Running Rollback!"

                            sh '''
                                ssh -o StrictHostKeyChecking=no -i $SSH_KEY $SSH_HOST '
                                    if [ -f $TAG_FILE ]; then
                                        ROLLBACK_TAG=$(cat $TAG_FILE)
                                        echo "Rolling back to previous version: $ROLLBACK_TAG"

                                        export TAG=$ROLLBACK_TAG
                                        docker-compose pull
                                        docker-compose up -d --force-recreate
                                    else
                                        echo "⚠️ No previous TAG found — rollback not possible"
                                    fi
                                '
                            '''

                            throw e
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment Successful! Version: $TAG"
        }
        failure {
            echo "❌ Deployment Failed — rollback executed (if possible)"
        }
    }
}
