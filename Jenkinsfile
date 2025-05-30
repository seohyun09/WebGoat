pipeline {
    agent any

    tools {
        jdk 'JDK_17'
        maven 'M3'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/seohyun09/WebGoat.git', branch: 'develop', credentialsId: 'none'
            }
        }

        stage('Build') {
            steps {
                script {
                    echo "Building the project..."
                    // FindSecBugs 분석을 위해 필요한 .class 파일을 생성하는 빌드 명령어
                    sh "mvn clean package"
                }
            }
        }

        stage('Static Analysis with FindSecBugs and S3 Upload') {
            steps {
                script {
                    echo "Calling Lambda to trigger FindSecBugs analysis on EC2 and upload results to S3..."

                    withCredentials([
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials-id' // Jenkins Credentials에 등록된 AWS 자격 증명 ID
                    ]) {
                        // Lambda 함수를 호출합니다.
                        // Lambda 함수는 이제 FindSecBugs 분석을 수행하고,
                        // 그 결과를 S3에 업로드한 후, 업로드된 S3 URI를 응답으로 반환해야 합니다.
                        sh """
                        aws lambda invoke \\
                            --function-name findsecbugs_lambda \\
                            --region ap-southeast-2 \\
                            --payload '{}' \\
                            /tmp/findsecbugs-lambda-output.json
                        """
                    }

                    echo "Lambda call completed. Checking output for S3 URI..."
                    def lambdaOutput = sh(returnStdout: true, script: "cat /tmp/findsecbugs-lambda-output.json").trim()
                    echo "Lambda Response: ${lambdaOutput}"

                    def s3ResultUri = null
                    try {
                        def jsonSlurper = new groovy.json.JsonSlurper()
                        def responseJson = jsonSlurper.parseText(lambdaOutput)
                        // Lambda 응답의 'body'가 JSON 문자열일 수 있으므로 다시 파싱
                        def responseBody = new groovy.json.JsonSlurper().parseText(responseJson.body)
                        s3ResultUri = responseBody.s3ResultUri // Lambda 응답의 실제 S3 URI 키
                    } catch (Exception e) {
                        echo "Failed to parse Lambda response or find 's3ResultUri': ${e.getMessage()}"
                    }


                    if (s3ResultUri) {
                        echo "✅ FindSecBugs results successfully uploaded to S3: ${s3ResultUri}"
                        // 선택 사항: Jenkins 빌드 아티팩트 링크로 S3 URI를 게시
                        // Jenkins Post-build Actions에서 "Publish HTML reports" 플러그인 등을 사용하여
                        // S3에 있는 HTML 보고서로 바로 링크를 걸 수 있습니다.
                    } else {
                        echo "❌ Could not retrieve S3 result URI from Lambda response. Check Lambda function's return format."
                        error "FindSecBugs S3 upload failed or result URI not found." // 빌드를 실패시킴
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ FindSecBugs analysis and S3 upload completed successfully!"
        }
        failure {
            echo "❌ FindSecBugs analysis or S3 upload failed. 로그를 확인하세요."
        }
        always {
            cleanWs()
            echo "Workspace cleaned."
        }
    }
}
