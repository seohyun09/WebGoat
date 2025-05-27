pipeline {
    agent { label 'sast-infer-agent' }

    tools {
        jdk 'JDK_17'
    }

    environment {
        // AWS_DEFAULT_REGION = 'ap-southeast-2' // 이제 필요 없으므로 주석 처리 또는 제거
        // ECR_REPO = '950846564115.dkr.ecr.ap-southeast-2.amazonaws.com/devops_ecr' // 제거
        // IMAGE_TAG = 'latest' // 제거
        // TASK_FAMILY = 'ecs_task_definition' // 제거
        // SERVICE_NAME = 'ecs_task_definition-service-6lc21cqh' // 제거
        // CLUSTER_NAME = 'ecs_cluster' // 제거
        // S3_BUCKET = 'soniaa-s3' // 제거
        // DEPLOY_APP = 'devops_codedeploy' // 제거
        // DEPLOY_GROUP = 'devops' // 제거
        // BUNDLE = 'deploy-bundle.zip' // 제거
        // CONTAINER_PORT = '9090' // 제거
        // EXECUTION_ROLE_ARN = 'arn:aws:iam::950846564115:role/iam_codedeploy' // 제거
        // AWS_CREDENTIALS_ID = 'db8c18b0-0e6d-4565-a0cc-2f9bb1405357' // 제거
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/WebGoat/WebGoat.git', credentialsId: 'none'
            }
        }

        stage('Static Analysis with Infer') {
            steps {
                script {
                    def workspacePath = pwd()

                    echo "Starting Infer static analysis for WebGoat..."
                    sh "docker run --rm -v ${workspacePath}:/src facebook/infer:latest infer -- mvn -f /src/webgoat-parent/pom.xml clean install"
                    echo "Infer analysis completed. Collecting results..."
                    
                    archiveArtifacts artifacts: 'infer-out/**', fingerprint: true, allowEmpty: true
                    
                    echo "--- Infer Analysis Report (report.json) ---"
                    sh "cat infer-out/report.json || echo 'infer-out/report.json not found or empty.'"
                    echo "------------------------------------------"
                }
            }
        }

        // --- 아래 스테이지들은 주석 처리하거나 삭제합니다 ---
        // stage('Build Docker Image') {
        //     steps {
        //         sh 'docker build -t $ECR_REPO:$IMAGE_TAG .'
        //     }
        // }

        // stage('Login to AWS ECR') {
        //     steps {
        //         withCredentials([
        //             usernamePassword(
        //                 credentialsId: env.AWS_CREDENTIALS_ID,
        //                 usernameVariable: 'AWS_ACCESS_KEY_ID',
        //                 passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        //             )
        //         ]) {
        //             sh '''
        //                 aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
        //                 aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
        //                 aws configure set region $AWS_DEFAULT_REGION

        //                 aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
        //                 docker login --username AWS --password-stdin $ECR_REPO
        //             '''
        //         }
        //     }
        // }

        // stage('Push to ECR') {
        //     steps {
        //         sh 'docker push $ECR_REPO:$IMAGE_TAG'
        //     }
        // }

        // stage('Deploy via CodeDeploy') {
        //     steps {
        //         sh '''
        //             aws deploy create-deployment \
        //                 --application-name $DEPLOY_APP \
        //                 --deployment-group-name $DEPLOY_GROUP \
        //                 --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
        //                 --s3-location bucket=$S3_BUCKET,bundleType=zip,key=$BUNDLE \
        //                 --region $AWS_DEFAULT_REGION
        //         '''
        //     }
        // }
    }

    post {
        success {
            echo "✅ Infer analysis completed successfully!"
        }
        failure {
            echo "❌ Infer analysis failed. Please check logs."
        }
        always {
            deleteDir()
        }
    }
}
