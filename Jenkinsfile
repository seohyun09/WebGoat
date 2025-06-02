pipeline {
    agent any

    tools {
        jdk 'JDK_17'
        maven 'M3'
    }

    environment {
        CODEQL_JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        CODEQL_EXTRACTOR_JAVA_ROOT = "/usr/local/bin/codeql/java"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/seohyun09/WebGoat.git', branch: 'develop'
            }
        }

        stage('CodeQL Create DB') {
            steps {
                sh '''
                    # CODEQL_JAVA_HOME이 올바르게 설정되었는지 확인
                    echo "JAVA_HOME: $JAVA_HOME"
                    echo "CODEQL_JAVA_HOME: $CODEQL_JAVA_HOME"
                    
                    # CodeQL CLI가 mvn 명령을 직접 실행하도록 유도
                    /usr/local/bin/codeql database create webgoat-db \\
                      --language=java \\
                      --command="mvn clean compile -DskipTests --release 17" \\
                      --no-autobuild  # <--- 이 옵션 추가
                '''
            }
        }

        stage('CodeQL Analyze') {
            steps {
                sh '''
                    /usr/local/bin/codeql database analyze webgoat-db \\
                      codeql-repo/java/ql/src/codeql-suites/java-code-scanning.qls \\
                      --format=sarifv2.1.0 \\
                      --output=webgoat-codeql-results.sarif
                '''
            }
        }
    }

    post {
        success {
            echo "✅ CodeQL 정적 분석 완료!"
        }
        failure {
            echo "❌ CodeQL 분석 실패. 로그를 확인하세요."
        }
        always {
            cleanWs()
            echo "Workspace cleaned."
        }
    }
}
