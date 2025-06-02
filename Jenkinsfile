pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-2'
        S3_BUCKET = 'codeql-bucket'
        LAMBDA_NAME = 'codeql_lambda'
        S3_SOURCE_KEY = 'source.zip'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/seohyun09/WebGoat.git', branch: 'develop'
            }
        }

        stage('Zip and Upload Source') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws_credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        echo "[📦] 소스코드 압축 중..."
                        zip -r source.zip . -x "*.git*" "*.idea*" "target/*"
                        echo "[☁️] S3에 업로드 중..."
                        aws s3 cp source.zip s3://$S3_BUCKET/$S3_SOURCE_KEY --region $AWS_REGION
                    '''
                }
            }
        }

        stage('Invoke Lambda') {
            steps {
                sh '''
                    echo "[🚀] Lambda 함수 호출 중..."
                    aws lambda invoke \
                        --function-name $LAMBDA_NAME \
                        --payload '{"s3_key":"'$S3_SOURCE_KEY'"}' \
                        --region $AWS_REGION \
                        --cli-binary-format raw-in-base64-out \
                        lambda_output.json

                    echo "[📄] Lambda 호출 응답:"
                    cat lambda_output.json
                '''
            }
        }

        stage('Download Results') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'codeql-aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        echo "[📥] 분석 결과 다운로드 중..."
                        aws s3 cp s3://$S3_BUCKET/webgoat/webgoat-codeql-results.sarif webgoat-codeql-results.sarif --region $AWS_REGION
                    '''
                }
            }
        }

        stage('Publish Report') {
            steps {
                publishHTML([
                    reportDir: '.',
                    reportFiles: 'webgoat-codeql-results.sarif',
                    reportName: 'CodeQL 분석 리포트',
                    keepAll: true,
                    alwaysLinkToLastBuild: true,
                    allowMissing: false
                ])
            }
        }
    }

    post {
        always {
            cleanWs()
            echo "작업 공간 정리 완료"
        }
    }
}
