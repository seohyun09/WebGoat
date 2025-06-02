pipeline {
    agent any

    tools {
        jdk 'JDK_17'
        maven 'M3'
    }

    environment {
        CODEQL_JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        CODEQL_EXTRACTOR_JAVA_ROOT = "/usr/local/bin/codeql/java"
        # CodeQL이 사용할 기본 자바를 명시적으로 설정 (JavacTool 문제 방지)
        # 이 환경변수는 CodeQL이 내부적으로 사용하는 javac를 제어할 수 있습니다.
        JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
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
                    # Maven Dependency Plugin을 사용하여 컴파일 클래스패스를 획득
                    # 이 명령어는 프로젝트의 모든 컴파일 종속성을 단일 경로 문자열로 출력합니다.
                    # 예를 들어, ':path/to/jar1.jar:path/to/jar2.jar'
                    # WebGoat 디렉토리 안으로 이동하여 실행
                    sh 'cd WebGoat && mvn dependency:build-classpath -Dmdep.outputFile=classpath.txt'
                    # 획득한 클래스패스를 Jenkins 변수에 저장
                    env.MAVEN_CLASSPATH = readFile('WebGoat/classpath.txt').trim()
                    echo "Maven Classpath: ${env.MAVEN_CLASSPATH}"
                }
            }
        }

        stage('CodeQL Create DB with Javac') {
            steps {
                sh '''
                    # 현재 작업 디렉토리를 WebGoat 프로젝트 루트로 변경
                    cd WebGoat
                    
                    echo "Using JAVA_HOME: $JAVA_HOME"
                    echo "Using MAVEN_CLASSPATH: ${MAVEN_CLASSPATH}"

                    # CodeQL CLI에 직접 javac 명령 전달
                    # -source 17, -target 17은 pom.xml과 일치시킴
                    # -cp "${MAVEN_CLASSPATH}" 는 Maven에서 가져온 클래스패스
                    # src/main/java 는 소스 코드 루트
                    # $(find src/main/java -name "*.java") 는 모든 Java 소스 파일을 찾음
                    /usr/local/bin/codeql database create webgoat-db \\
                      --language=java \\
                      --command="javac -source 17 -target 17 -cp "${MAVEN_CLASSPATH}" $(find src/main/java -name "*.java")" \\
                      --no-autobuild
                '''
            }
        }

        stage('CodeQL Analyze') {
            steps {
                sh '''
                    cd WebGoat # 분석을 위해 프로젝트 디렉토리로 이동
                    /usr/local/bin/codeql database analyze webgoat-db \\
                      /usr/local/bin/codeql/ql/java/ql/src/codeql-suites/java-code-scanning.qls \\
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
