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
        
        stage ('deploy to EC2') {
            steps {
                script {
                    echo "deploying to shell-script to ec2"
                    def shellCmd = "bash ./php.sh"
                    sshagent ([SSH_CREDENTIALS_ID]) {
                        sh "scp -o StrictHostKeyChecking=no php.sh ${REMOTE_USER}@${EC2_IP}:/home/ubuntu"
                        sh "ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${EC2_IP} ${shellCmd}"
                    }
                }
            }
        }
    }
}