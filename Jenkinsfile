pipeline {
    agent any

    tools {
        jdk 'JDK_17'
        maven 'M3'
    }

    environment {
        CODEQL_JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        CODEQL_EXTRACTOR_JAVA_ROOT = "/usr/local/bin/codeql/java"
        JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        
        # 새로운 환경 변수 추가: CodeQL의 자동 빌드 로직을 추가적으로 비활성화 시도
        # 특히 Spring 관련 빌드 감지를 억제하여 불필요한 Maven 호출을 줄일 수 있습니다.
        CODEQL_AUTOBUILD_JAVA_SPRING_DISABLED = "true"
        
        # CodeQL이 Maven 워퍼(wrapper)를 사용하는 것을 명시적으로 비활성화하여
        # CodeQL이 './mvnw' 대신 시스템에 설치된 'mvn'을 사용하도록 할 수 있습니다.
        # 이 시도도 도움이 될 수 있습니다.
        CODEQL_EXTRACTOR_JAVA_RUN_MVNW = "false"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/seohyun09/WebGoat.git', branch: 'develop'
            }
        }

        stage('Prepare Maven Classpath') {
            steps {
                script {
                    dir('WebGoat') {
                        echo "Running Maven dependency:build-classpath to get compilation classpath..."
                        sh 'mvn dependency:build-classpath -Dmdep.outputFile=classpath.txt'
                        env.MAVEN_CLASSPATH = readFile('classpath.txt').trim()
                        echo "Maven Classpath prepared: ${env.MAVEN_CLASSPATH}"
                    }
                }
            }
        }

        stage('CodeQL Create DB with Javac') {
            steps {
                sh '''
                    cd WebGoat
                    
                    echo "Current JAVA_HOME for CodeQL: $JAVA_HOME"
                    echo "Using MAVEN_CLASSPATH for compilation: ${MAVEN_CLASSPATH}"
                    echo "CODEQL_AUTOBUILD_JAVA_SPRING_DISABLED: ${CODEQL_AUTOBUILD_JAVA_SPRING_DISABLED}"
                    echo "CODEQL_EXTRACTOR_JAVA_RUN_MVNW: ${CODEQL_EXTRACTOR_JAVA_RUN_MVNW}"

                    # CodeQL CLI에 직접 javac 명령 전달
                    # 핵심 변경: javac의 절대 경로를 사용하여 어떤 javac가 실행될지 명확히 합니다.
                    # 이 경로($JAVA_HOME/bin/javac)는 Jenkins 에이전트에 Corretto 17이 설치된 경로와 일치해야 합니다.
                    /usr/local/bin/codeql database create webgoat-db \\
                      --language=java \\
                      --command="${JAVA_HOME}/bin/javac -source 17 -target 17 -cp "${MAVEN_CLASSPATH}" $(find src/main/java -name "*.java")" \\
                      --no-autobuild
                '''
            }
        }

        stage('CodeQL Analyze') {
            steps {
                dir('WebGoat') {
                    /usr/local/bin/codeql database analyze webgoat-db \\
                      /usr/local/bin/codeql/ql/java/ql/src/codeql-suites/java-code-scanning.qls \\
                      --format=sarifv2.1.0 \\
                      --output=webgoat-codeql-results.sarif
                }
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
