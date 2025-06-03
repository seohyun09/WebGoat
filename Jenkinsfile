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
                script {
                    echo "[🚀] Lambda 함수 호출 중..."
                    def lambdaResult = sh(script: """
                        aws lambda invoke \\
                            --function-name ${env.LAMBDA_NAME} \\
                            --payload '{"s3_key":"${env.S3_SOURCE_KEY}"}' \\
                            --region ${env.AWS_REGION} \\
                            --cli-binary-format raw-in-base64-out \\
                            /dev/stdout | cat
                    """, returnStdout: true).trim()
        
                    echo "[📄] Lambda 호출 응답:\n${lambdaResult}"
        
                    def jsonOutput = readJSON text: lambdaResult
                    def lambdaStatusCode = jsonOutput.statusCode
                    def ssmCommandStatus = jsonOutput.body ? readJSON(text: jsonOutput.body).Status : null
        
                    if (lambdaStatusCode != 200) {
                        error "Lambda 함수 호출 실패: 상태 코드 ${lambdaStatusCode}"
                    } else if (ssmCommandStatus == 'Failed' || ssmCommandStatus == 'Cancelled' || ssmCommandStatus == 'TimedOut') {
                        error "EC2에서 CodeQL 분석 실패 또는 타임아웃됨. SSM Command Status: ${ssmCommandStatus}\nOutput: ${jsonOutput.body}"
                    } else if (ssmCommandStatus != 'Success') {
                        // 예상치 못한 상태 (예: InProgress인데 Lambda가 일찍 종료된 경우 등)
                        error "Lambda 함수 호출이 예상치 못한 상태로 종료됨: ${ssmCommandStatus}\nOutput: ${jsonOutput.body}"
                    }
                    echo "Lambda 함수 및 CodeQL 분석 성공!"
                }
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
