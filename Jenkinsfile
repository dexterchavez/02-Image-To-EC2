pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = "ap-southeast-1"
        REPO_NAME   = "petmed"
        ACCOUNT_ID  = "368166794913"
        IMAGE_TAG   = "${BUILD_NUMBER}"
        ECR_URI     = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/dexterchavez/petmed.git',
                    credentialsId: 'PAT-Security-Tokens'
            }
        }

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

        stage('Build and Push Docker Image') {
            steps {
                sh '''
                    echo "🔨 Building Docker image with tag: $IMAGE_TAG"
                    docker build --platform linux/amd64 -t $REPO_NAME:$IMAGE_TAG .
                    
                    echo "🏷️ Tagging image for ECR"
                    docker tag $REPO_NAME:$IMAGE_TAG $ECR_URI:$IMAGE_TAG
                    
                    echo "📤 Pushing image to ECR"
                    docker push $ECR_URI:$IMAGE_TAG
                '''
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up..."
            sh 'docker system prune -f || true'
        }
        failure {
            echo "❌ Build or push failed!"
        }
        success {
            echo "✅ Image pushed successfully: $ECR_URI:$IMAGE_TAG"
        }
    }
}
