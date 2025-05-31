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
                    sh "mvn clean package"
                }
            }
        }

        stage('Static Analysis with FindSecBugs and S3 Upload') {
            steps {
                script {
                    echo "Calling Lambda to trigger FindSecBugs analysis on EC2 and upload results to S3..."

                    // Jenkins에 등록한 시크릿 텍스트 자격증명 ID를 사용
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh """
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
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
                        def responseBody = new groovy.json.JsonSlurper().parseText(responseJson.body)
                        s3ResultUri = responseBody.s3ResultUri
                    } catch (Exception e) {
                        echo "Failed to parse Lambda response or find 's3ResultUri': ${e.getMessage()}"
                    }

                    if (s3ResultUri) {
                        echo "✅ FindSecBugs results successfully uploaded to S3: ${s3ResultUri}"
                    } else {
                        echo "❌ Could not retrieve S3 result URI from Lambda response. Check Lambda function's return format."
                        error "FindSecBugs S3 upload failed or result URI not found."
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
