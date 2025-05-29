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

        stage('Static Analysis with FindSecBugs') {
            steps {
                script {
                    echo "Calling Lambda to trigger FindSecBugs analysis on EC2..."

                    sh """
                    aws lambda invoke \
                      --function-name findsecbugs_lambda \
                      --region ap-northeast-2 \
                      --payload '{}' \
                      /tmp/findsecbugs-lambda-output.json
                    """

                    echo "Lambda call completed. Output:"
                    sh "cat /tmp/findsecbugs-lambda-output.json"
                }
            }
            post {
                always {
                    recordIssues enabledForFailure: true, tools: [
                        spotBugs(pattern: '**/target/spotbugsXml.xml')
                    ]
                }
            }
        }
    }

    post {
        success {
            echo "✅ FindSecBugs analysis completed successfully!"
        }
        failure {
            echo "❌ FindSecBugs analysis failed. 로그 확인 필요."
        }
        always {
            cleanWs()
            echo "Workspace cleaned."
        }
    }
}
