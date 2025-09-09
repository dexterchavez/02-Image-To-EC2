pipeline {
    agent any

    environment {
        ACCOUNT_ID         = "368166794913"
        AWS_DEFAULT_REGION = "ap-southeast-1"
        REPO_NAME          = "petmed"
        EC2_IP             = "13.212.2.119"
        ECR_URI            = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}"
        GITHUB_REPO        = "https://github.com/dexterchavez/02-Image-To-EC2.git"
        REMOTE_USER        = "ubuntu"
        SSH_CREDENTIALS_ID = "ubuntu-mrdexterchavez"
    }

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'ECR image tag to deploy (must exist in ECR)')
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

        stage('prepare deploy script') {
            steps {
                script {
                    echo "📝 Creating deploy.sh file..."
                    writeFile file: 'deploy.sh', text: """
                        #!/bin/bash
                        set -e

                        echo "✅ Running deploy script on EC2..."

                        # Install AWS CLI if not present
                        if ! command -v aws &> /dev/null; then
                          echo "Installing AWS CLI..."
                          curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                          unzip -q awscliv2.zip
                          sudo ./aws/install
                        fi

                        # Authenticate Docker to ECR
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} \\
                          | sudo docker login --username AWS --password-stdin ${ECR_URI}

                        echo "📦 Pulling image: ${ECR_URI}:${params.IMAGE_TAG}"
                        sudo docker pull ${ECR_URI}:${params.IMAGE_TAG}

                        echo "🛑 Stopping old container (if running)..."
                        sudo docker stop ${REPO_NAME} || true
                        sudo docker rm ${REPO_NAME} || true

                        echo "🚀 Running new container..."
                        sudo docker run -d --name ${REPO_NAME} -p 80:80 ${ECR_URI}:${params.IMAGE_TAG}
                    """
                }
            }
        }

        stage('deploy to EC2') {
            steps {
                sshagent([env.SSH_CREDENTIALS_ID]) {
                    sh """
                        echo "🚀 Deploying app to EC2..."
                        scp -o StrictHostKeyChecking=no deploy.sh ${REMOTE_USER}@${EC2_IP}:/home/${REMOTE_USER}/
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${EC2_IP} 'chmod +x /home/${REMOTE_USER}/deploy.sh && /home/${REMOTE_USER}/deploy.sh'
                    """
                }
            }
        }
    }
}
