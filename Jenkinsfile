pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = "ap-southeast-1"
        REPO_NAME   = "petmed"
        ACCOUNT_ID  = "368166794913"
        ECR_URI     = "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}"
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

        stage('Get Latest Image Tag') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'AWS-Credentials']]) {
                    script {
                        def latestTag = sh(
                            script: """
                                aws ecr describe-images \
                                  --repository-name $REPO_NAME \
                                  --region $AWS_DEFAULT_REGION \
                                  --query 'sort_by(imageDetails,& imagePushedAt)[-1].imageTags[0]' \
                                  --output text
                            """,
                            returnStdout: true
                        ).trim()
                        echo "📌 Latest image tag found: ${latestTag}"
                        env.IMAGE_TAG = latestTag
                    }
                }
            }
        }

        stage('Pull Image from ECR') {
            steps {
                sh '''
                    echo "📥 Pulling image $ECR_URI:${IMAGE_TAG} ..."
                    docker pull $ECR_URI:${IMAGE_TAG}
                '''
            }
        }

        stage('Run Docker Container') {
            steps {
                sh '''
                    echo "🚀 Running container from image: $ECR_URI:${IMAGE_TAG}"
                    docker rm -f petmed || true
                    docker run -d --name petmed -p 80:80 $ECR_URI:${IMAGE_TAG}
                '''
            }
        }
    }

    post {
        failure {
            echo "❌ Failed to pull or run container!"
        }
        success {
            echo "✅ Successfully deployed: $ECR_URI:${IMAGE_TAG}"
        }
    }
}
