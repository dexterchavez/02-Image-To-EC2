pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = "ap-southeast-1"
        REPO_NAME   = "petmed"
        ACCOUNT_ID  = "368166794913"
        IMAGE_TAG   = "${BUILD_NUMBER}"   // Or set to "latest" if you prefer
        ECR_URI     = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}"
        CONTAINER_NAME = "petmed-container"
    }

    stages {
        stage('Login to AWS ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
                                  credentialsId: 'AWS-Credentials']]) {
                    sh '''
                        echo "🔑 Logging in to AWS ECR..."
                        aws sts get-caller-identity
                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
                          docker login --username AWS --password-stdin $ECR_URI
                    '''
                }
            }
        }

        stage('Pull Image from ECR') {
            steps {
                sh '''
                    echo "📥 Pulling image $ECR_URI:$IMAGE_TAG ..."
                    docker pull $ECR_URI:$IMAGE_TAG
                '''
            }
        }

        stage('Run Docker Container') {
            steps {
                sh '''
                    echo "🛑 Stopping old container (if exists)..."
                    docker rm -f $CONTAINER_NAME || true

                    echo "🚀 Running new container from image..."
                    docker run -d --name $CONTAINER_NAME -p 80:80 $ECR_URI:$IMAGE_TAG
                '''
            }
        }
    }

    post {
        failure {
            echo "❌ Failed to pull or run container!"
        }
        success {
            echo "✅ Container is running from: $ECR_URI:$IMAGE_TAG"
        }
    }
}
