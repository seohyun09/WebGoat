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

                    withCredentials([[ 
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials-id'
                    ]]) {
                        sh """
                        aws lambda invoke \
                          --function-name findsecbugs_lambda \
                          --region ap-southeast-2 \
                          --payload '{}' \
                          /tmp/findsecbugs-lambda-output.json
                        """
                    }

                    echo "Lambda call completed. Output:"
                    sh "cat /tmp/findsecbugs-lambda-output.json"
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
