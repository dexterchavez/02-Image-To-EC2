pipeline {
    agent any

    environment {
        ACCOUNT_ID            = "368166794913"
        AWS_DEFAULT_REGION    = "ap-southeast-1"
        REPO_NAME             = "petmed"
        EC2_IP                = '13.212.2.119'
        ECR_URI               = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}"
        GITHUB_REPO           = 'https://github.com/dexterchavez/02-Image-To-EC2.git'
        REMOTE_USER           = "ubuntu"
        SSH_CREDENTIALS_ID    = "mrdexterchavez"
    }

    stages {
        stage ('fetch code') {
            steps {
                script {
                    echo "Pull source code from Git"
                    git branch: 'main', url: "${GITHUB_REPO}"
                }
            }
        }
        
        stage('prepare deploy script') {
            steps {
                sh '''
                    echo "📝 Creating deploy.sh file..."
                    cat > deploy.sh <<'EOF'
                    #!/bin/bash
                    echo "✅ Running deploy script on EC2..."
                    sudo apt-get update -y
                    sudo apt-get install -y docker.io
                    # add more steps here if needed
                    EOF
                    chmod +x deploy.sh
                '''
            }
        }

        stage('deploy to EC2') {
            steps {
                sshagent(['ubuntu-mrdexterchavez']) {
                    sh '''
                        echo "🚀 Deploying app to EC2..."

                        # Instead of php.sh, either:
                        # - copy deploy.sh (if you’re using Docker deployment)
                        # - or run inline commands directly

                        scp -o StrictHostKeyChecking=no deploy.sh ubuntu@${EC2_IP}:/home/ubuntu
                        ssh -o StrictHostKeyChecking=no ubuntu@${EC2_IP} "chmod +x /home/ubuntu/deploy.sh && /home/ubuntu/deploy.sh"
                    '''
                }
            }
        }

    }
}