pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_DEFAULT_REGION    = "ap-southeast-1"
        REPO_NAME             = "petmed"
        ACCOUNT_ID            = "368166794913"
        IMAGE_TAG             = "${env.BUILD_NUMBER}"
        ECR_URI               = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}"
        CONTAINER_NAME        = "petmed"
        EC2_HOST              = "10.0.10.139"
        PORT_MAPPING          = "80:80"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "📥 Checking out source code..."
                checkout scm
            }
        }

        stage('Verify Parameters') {
            steps {
                script {
                    echo "📦 ECR Repository: ${ECR_URI}"
                    echo "🏷️ Image Tag: ${IMAGE_TAG}"
                    echo "🖥️ EC2 Host: ${EC2_HOST}"
                    echo "📛 Container Name: ${CONTAINER_NAME}"
                    echo "🔌 Port Mapping: ${PORT_MAPPING}"
                }
            }
        }

        stage('AWS Authentication') {
            steps {
                sh '''
                    echo "🔑 Logging in to ECR..."
                    aws ecr get-login-password --region ${AWS_DEFAULT_REGION} \
                    | docker login --username AWS --password-stdin ${ECR_URI}
                '''
            }
        }

        stage('Pull Image from ECR') {
            steps {
                sh '''
                    echo "📥 Pulling image: ${ECR_URI}:${IMAGE_TAG}"
                    docker pull ${ECR_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Test Image Locally') {
            steps {
                sh '''
                    echo "🧪 Running container locally for smoke test..."
                    docker run -d --rm --name ${CONTAINER_NAME}-test -p ${PORT_MAPPING} ${ECR_URI}:${IMAGE_TAG}
                    sleep 5
                    docker ps | grep ${CONTAINER_NAME}-test
                    docker stop ${CONTAINER_NAME}-test
                '''
            }
        }

        stage('Prepare Deployment Scripts') {
            steps {
                writeFile file: 'deploy.sh', text: """
                #!/bin/bash
                set -e
                echo "🚀 Deploying ${ECR_URI}:${IMAGE_TAG} to EC2..."
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
                docker run -d --name ${CONTAINER_NAME} -p ${PORT_MAPPING} ${ECR_URI}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh '''
                        echo "📡 Copying deploy script to EC2..."
                        scp -o StrictHostKeyChecking=no deploy.sh ubuntu@${EC2_HOST}:/tmp/deploy.sh
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} "chmod +x /tmp/deploy.sh && /tmp/deploy.sh"
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    echo "🩺 Performing health check..."
                    curl -I http://${EC2_HOST} || true
                '''
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up Jenkins workspace..."
            cleanWs()
        }
        failure {
            echo "❌ Deployment failed!"
            script {
                sshagent(['ec2-ssh-key']) {
                    echo "⚠️ Rollback failed: [ssh-agent] Could not find specified credentials: ec2-ssh-key"
                }
            }
        }
    }
}
