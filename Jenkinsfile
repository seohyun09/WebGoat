pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-southeast-2'
        ECR_REPO = '950846564115.dkr.ecr.ap-southeast-2.amazonaws.com/devops_ecr'
        IMAGE_TAG = 'latest'
        TASK_FAMILY = 'ecs_task_definition'
        SERVICE_NAME = 'ecs_task_definition-service-6lc21cqh'
        CLUSTER_NAME = 'ecs_cluster'
        S3_BUCKET = 'soniaa-09'
        DEPLOY_APP = 'devops_codedeploy'
        DEPLOY_GROUP = 'devops'
        BUNDLE = 'deploy-bundle.zip'
        CONTAINER_PORT = '8080' // 실제 앱 포트에 맞게 수정
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $ECR_REPO:$IMAGE_TAG .'
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
                sh 'docker push $ECR_REPO:$IMAGE_TAG'
            }
        }

        stage('Generate taskdef.json') {
            steps {
                script {
                    def taskdef = """{
  "family": "${TASK_FAMILY}",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "devops-container",
      "image": "${ECR_REPO}:${IMAGE_TAG}",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "containerPort": ${CONTAINER_PORT},
          "protocol": "tcp"
        }
      ]
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "${EXECUTION_ROLE_ARN}"
}"""
                    writeFile file: 'taskdef.json', text: taskdef
                }
            }
        }

        stage('Generate appspec.yaml') {
            steps {
                script {
                    def taskDefArn = sh(
                        script: "aws ecs register-task-definition --cli-input-json file://taskdef.json --query 'taskDefinition.taskDefinitionArn' --output text --region $AWS_DEFAULT_REGION",
                        returnStdout: true
                    ).trim()

                    def appspec = """version: 1
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "${taskDefArn}"
        LoadBalancerInfo:
          ContainerName: "devops-container"
          ContainerPort: ${CONTAINER_PORT}
"""
                    writeFile file: 'appspec.yaml', text: appspec
                }
            }
        }

        stage('Bundle for CodeDeploy') {
            steps {
                sh 'zip -r $BUNDLE appspec.yaml taskdef.json'
            }
        }

        stage('Deploy via CodeDeploy') {
            steps {
                sh '''
                    aws s3 cp $BUNDLE s3://$S3_BUCKET/$BUNDLE --region $AWS_DEFAULT_REGION

                    aws deploy create-deployment \
                        --application-name $DEPLOY_APP \
                        --deployment-group-name $DEPLOY_GROUP \
                        --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
                        --s3-location bucket=$S3_BUCKET,bundleType=zip,key=$BUNDLE \
                        --region $AWS_DEFAULT_REGION
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Build and deployment completed successfully!"
        }
        failure {
            echo "❌ Build or deployment failed. Please check logs."
        }
    }
}
