pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-southeast-2'
        ECR_REPO = '950846564115.dkr.ecr.ap-southeast-2.amazonaws.com/devops_ecr'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t $ECR_REPO:latest .'
                }
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'db8c18b0-0e6d-4565-a0cc-2f9bb1405357',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set region $AWS_DEFAULT_REGION

                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
                        docker login --username AWS --password-stdin $ECR_REPO
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh '''
                        docker push $ECR_REPO:latest
                    '''
                }
            }
        }
    }
}
