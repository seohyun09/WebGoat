pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-2'
        ECR_URI = '950846564115.dkr.ecr.ap-southeast-2.amazonaws.com'
        REPOSITORY = 'devops_ecr'
        IMAGE_TAG = 'latest'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${REPOSITORY}:latest .'
                sh 'docker tag ${REPOSITORY}:latest ${ECR_URI}/${REPOSITORY}:${IMAGE_TAG}'
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'devops_ecr_plugin', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                        export AWS_DEFAULT_REGION=$AWS_REGION
        
                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin 950846564115.dkr.ecr.ap-southeast-2.amazonaws.com
                    '''
                }
            }
        }


        stage('Push to ECR') {
            steps {
                sh 'docker push $ECR_REPO:$IMAGE_TAG'
            }
        }
    }
}
