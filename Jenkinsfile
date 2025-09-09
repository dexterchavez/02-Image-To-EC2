pipeline {
    agent any
    
    parameters {
        string(name: 'ECR_REPOSITORY_URI', defaultValue: '368166794913.dkr.ecr.ap-southeast-1.amazonaws.com/petmed', description: 'ECR Repository URI')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag to deploy')
        string(name: 'EC2_HOST', defaultValue: '10.0.10.139', description: 'EC2 instance IP or hostname')
        string(name: 'EC2_USER', defaultValue: 'ubuntu', description: 'EC2 SSH user')
        string(name: 'CONTAINER_NAME', defaultValue: 'petmed', description: 'Container name')
        string(name: 'CONTAINER_PORT', defaultValue: '80', description: 'Container port')
        string(name: 'HOST_PORT', defaultValue: '80', description: 'Host port to map to container')
    }
    
    environment {
        AWS_DEFAULT_REGION = 'ap-southeast-1'
        AWS_ACCOUNT_ID     = '368166794913'
        ECR_REGISTRY       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '📥 Checking out source code...'
                // checkout scm   // uncomment if you actually want to clone repo code
            }
        }
        
        stage('Verify Parameters') {
            steps {
                script {
                    echo "📦 ECR Repository: ${params.ECR_REPOSITORY_URI}"
                    echo "🏷️ Image Tag: ${params.IMAGE_TAG}"
                    echo "🖥️ EC2 Host: ${params.EC2_HOST}"
                    echo "📛 Container Name: ${params.CONTAINER_NAME}"
                    echo "🔌 Port Mapping: ${params.HOST_PORT}:${params.CONTAINER_PORT}"
                }
            }
        }
        
        stage('AWS Authentication') {
            steps {
                script {
                    echo '🔑 Authenticating with AWS ECR...'
                    sh """
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} \
                        | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                }
            }
        }
        
        stage('Pull Image from ECR') {
            steps {
                script {
                    echo "📥 Pulling image from ECR..."
                    sh """
                        docker pull ${params.ECR_REPOSITORY_URI}:${params.IMAGE_TAG}
                    """
                }
            }
        }
        
        stage('Test Image Locally') {
            steps {
                script {
                    echo '🧪 Testing image locally...'
                    sh """
                        docker run --rm ${params.ECR_REPOSITORY_URI}:${params.IMAGE_TAG} --version || \
                        docker run --rm ${params.ECR_REPOSITORY_URI}:${params.IMAGE_TAG} echo "Image test successful"
                    """
                }
            }
        }
        
        stage('Prepare Deployment Scripts') {
            steps {
                script {
                    echo '📝 Preparing deployment & rollback scripts...'
                    
                    writeFile file: 'deploy.sh', text: """#!/bin/bash
set -e

CONTAINER_NAME="${params.CONTAINER_NAME}"
IMAGE_URI="${params.ECR_REPOSITORY_URI}:${params.IMAGE_TAG}"
HOST_PORT="${params.HOST_PORT}"
CONTAINER_PORT="${params.CONTAINER_PORT}"
AWS_REGION="${AWS_DEFAULT_REGION}"

echo "Starting deployment process..."

# Authenticate with ECR
aws ecr get-login-password --region \$AWS_REGION | docker login --username AWS --password-stdin ${ECR_REGISTRY}

# Stop & remove existing container
docker stop \$CONTAINER_NAME || true
docker rm \$CONTAINER_NAME || true

# Clean up old images
docker images ${params.ECR_REPOSITORY_URI} --format "table {{.Repository}}:{{.Tag}}" | grep -v \$IMAGE_URI | grep -v "REPOSITORY:TAG" | xargs -r docker rmi || true

# Pull latest image
docker pull \$IMAGE_URI

# Run new container
docker run -d \\
    --name \$CONTAINER_NAME \\
    --restart unless-stopped \\
    -p \$HOST_PORT:\$CONTAINER_PORT \\
    \$IMAGE_URI

sleep 5
if docker ps | grep -q \$CONTAINER_NAME; then
    echo "✅ Container \$CONTAINER_NAME is running"
else
    echo "❌ Failed to start container"
    docker logs \$CONTAINER_NAME
    exit 1
fi

echo "🚀 Deployment completed!"
"""
                    
                    writeFile file: 'rollback.sh', text: """#!/bin/bash
set -e
CONTAINER_NAME="${params.CONTAINER_NAME}"

echo "Starting rollback..."
docker stop \$CONTAINER_NAME || true
docker rm \$CONTAINER_NAME || true
echo "Rollback complete. Please redeploy previous version manually."
"""
                }
            }
        }
        
        stage('Deploy to EC2') {
            steps {
                script {
                    echo "🚀 Deploying to EC2 instance: ${params.EC2_HOST}"
                    sshagent(credentials: ['ec2-ssh-key']) {
                        sh """
                            scp -o StrictHostKeyChecking=no deploy.sh ${params.EC2_USER}@${params.EC2_HOST}:/tmp/
                            scp -o StrictHostKeyChecking=no rollback.sh ${params.EC2_USER}@${params.EC2_HOST}:/tmp/
                            ssh -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} '
                                chmod +x /tmp/deploy.sh /tmp/rollback.sh
                                sudo /tmp/deploy.sh
                            '
                        """
                    }
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo '🩺 Performing health check...'
                    sleep(time: 30, unit: 'SECONDS')
                    sshagent(credentials: ['ec2-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} '
                                if docker ps | grep -q ${params.CONTAINER_NAME}; then
                                    echo "✅ Container is running"
                                else
                                    echo "❌ Container is not running"
                                    docker logs ${params.CONTAINER_NAME}
                                    exit 1
                                fi
                            '
                        """
                    }
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    echo '🧹 Cleaning up local workspace...'
                    sh '''
                        rm -f deploy.sh rollback.sh
                        docker system prune -f --volumes || true
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '🎉 Deployment completed successfully!'
        }
        failure {
            echo '❌ Deployment failed!'
            script {
                try {
                    sshagent(credentials: ['ec2-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${params.EC2_USER}@${params.EC2_HOST} '
                                sudo /tmp/rollback.sh
                            '
                        """
                    }
                } catch (Exception e) {
                    echo "⚠️ Rollback failed: ${e.message}"
                }
            }
        }
        alway
