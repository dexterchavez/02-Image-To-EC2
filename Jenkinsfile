pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = "ap-southeast-1"
        REPO_NAME   = "petmed"
        ACCOUNT_ID  = "368166794913"
        ECR_URI     = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}"
        EC2_HOST    = "ubuntu@<EC2_PUBLIC_IP>"   // 🔑 replace with your EC2 public IP or DNS
    }

    stages {
        stage('Deploy to EC2') {
            steps {
                sshagent(['EC2-SSH-Key']) { // 🔑 Jenkins credential with your EC2 PEM key
                    sh """
                        ssh -o StrictHostKeyChecking=no $EC2_HOST '
                            echo "🔑 Logging in to AWS ECR..."
                            aws sts get-caller-identity

                            aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
                                docker login --username AWS --password-stdin $ECR_URI

                            echo "📌 Fetching latest image tag..."
                            IMAGE_TAG=$(aws ecr describe-images \
                                --repository-name $REPO_NAME \
                                --region $AWS_DEFAULT_REGION \
                                --query "sort_by(imageDetails,& imagePushedAt)[-1].imageTags[0]" \
                                --output text)

                            echo "📥 Pulling image: $ECR_URI:\$IMAGE_TAG"
                            docker pull $ECR_URI:\$IMAGE_TAG

                            echo "🚀 Running container..."
                            docker rm -f petmed || true
                            docker run -d --name petmed -p 80:80 $ECR_URI:\$IMAGE_TAG
                        '
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Deployment failed!"
        }
        success {
            echo "✅ Deployment successful on EC2: $ECR_URI"
        }
    }
}
