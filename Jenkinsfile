pipeline {
    agent any

    environment {
        ACCOUNT_ID         = "368166794913"
        AWS_DEFAULT_REGION = "ap-southeast-1"
        REPO_NAME          = "petmed"
        EC2_IP             = "13.212.2.119"
        GITHUB_REPO        = "https://github.com/dexterchavez/02-Image-To-EC2.git"
        REMOTE_USER        = "ubuntu"
        SSH_CREDENTIALS_ID = "ubuntu-mrdexterchavez"
        ECR_URI            = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}"
        IMAGE_TAG          = "${BUILD_NUMBER}"
    }

    stages {
        stage('fetch code') {
            steps {
                script {
                    echo "📥 Pulling source code from GitHub..."
                    git branch: 'main', url: "${GITHUB_REPO}"
                }
            }
        }

        stage('build & push image') {
            steps {
                script {
                    echo "🐳 Building and pushing Docker image to ECR..."

                    sh '''
                        # Authenticate with ECR
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_URI}

                        # Build Docker image (force amd64 for EC2 compatibility)
                        docker buildx build --platform linux/amd64 -t ${ECR_URI}:${IMAGE_TAG} -t ${ECR_URI}:latest .

                        # Push both tags
                        docker push ${ECR_URI}:${IMAGE_TAG}
                        docker push ${ECR_URI}:latest
                    '''
                }
            }
        }

        stage('prepare deploy script') {
            steps {
                script {
                    echo "📝 Creating deploy.sh file..."
                    writeFile file: 'deploy.sh', text: """#!/bin/bash
echo "✅ Running deploy script on EC2..."

# Install Docker if missing
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=\$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable docker
sudo systemctl start docker

# Authenticate with ECR
aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | sudo docker login --username AWS --password-stdin ${ECR_URI}

# Pull build-specific tag
sudo docker pull ${ECR_URI}:${IMAGE_TAG}

# Stop & remove old container
sudo docker stop petmed || true
sudo docker rm petmed || true

# Run new container
sudo docker run -d --name petmed -p 80:80 ${ECR_URI}:${IMAGE_TAG}
"""
                }
            }
        }

        stage('deploy to EC2') {
            steps {
                sshagent([SSH_CREDENTIALS_ID]) {
                    sh '''
                        echo "🚀 Deploying app to EC2..."
                        scp -o StrictHostKeyChecking=no deploy.sh ${REMOTE_USER}@${EC2_IP}:/home/${REMOTE_USER}/
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${EC2_IP} "chmod +x /home/${REMOTE_USER}/deploy.sh && /home/${REMOTE_USER}/deploy.sh"
                    '''
                }
            }
        }
    }
}
