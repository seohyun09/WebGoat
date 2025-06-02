pipeline {
    agent any

    tools {
        jdk 'JDK_17'
        maven 'M3'
    }

    environment {
        JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        CODEQL_JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        CODEQL_EXTRACTOR_JAVA_ROOT = "/usr/local/bin/codeql/java"
        CODEQL_AUTOBUILD_JAVA_SPRING_DISABLED = "true"
        CODEQL_EXTRACTOR_JAVA_RUN_MVNW = "false"
        PYTHONPATH = "/var/lib/jenkins/sarif-tools"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/seohyun09/WebGoat.git', branch: 'develop'
            }
        }

        stage('Prepare Maven Classpath') {
            steps {
                dir('WebGoat') {
                    sh 'mvn dependency:build-classpath -Dmdep.outputFile=classpath.txt'
                }
            }
        }

        stage('Create CodeQL Database') {
            steps {
                dir('WebGoat') {
                    script {
                        def classpath = readFile('classpath.txt').trim()
                        sh """
                            /usr/local/bin/codeql database create webgoat-db \\
                              --language=java \\
                              --command="\${JAVA_HOME}/bin/javac -source 17 -target 17 -cp '${classpath}' \$(find src/main/java -name '*.java')" \\
                              --no-autobuild
                        """
                    }
                }
            }
        }

        stage('CodeQL Analyze') {
            steps {
                dir('WebGoat') {
                    sh '''
                        /usr/local/bin/codeql database analyze webgoat-db \
                          /usr/local/bin/codeql/ql/java/ql/src/codeql-suites/java-code-scanning.qls \
                          --format=sarifv2.1.0 \
                          --output=webgoat-codeql-results.sarif
                    '''
                }
            }
        }

        stage('Generate & Publish CodeQL Report') {
            steps {
                dir('WebGoat') {
                    sh '''
                        mkdir -p codeql-html
                        python3 -m sarif.tools.sarif_to_html webgoat-codeql-results.sarif > codeql-html/index.html
                    '''
                    publishHTML(target: [
                        reportName: 'CodeQL Report',
                        reportDir: 'codeql-html',
                        reportFiles: 'index.html',
                        keepAll: true,
                        alwaysLinkToLastBuild: true,
                        allowMissing: false
                    ])
                }
            }
        }
    }

    post {
        success {
            echo "✅ CodeQL 정적 분석 및 리포트 생성 완료!"
        }
        failure {
            echo "❌ CodeQL 분석 실패. 로그 확인 필요."
        }
        always {
            dir('WebGoat') {
                sh 'rm -rf webgoat-db codeql-html webgoat-codeql-results.sarif'
            }
            cleanWs()
            echo "📦 정리 완료"
        }
    }
}
