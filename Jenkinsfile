pipeline {
    agent any // 젠킨스 에이전트가 어떤 노드에서든 실행되도록 설정

    tools {
        jdk 'JDK_17' // Jenkins 툴 설정에서 JDK 17 사용 명시
        maven 'M3'   // Jenkins 툴 설정에서 Maven 3 사용 명시
    }

    environment {
        # CodeQL이 Java 프로젝트를 분석할 때 사용할 JDK 경로를 명시적으로 설정합니다.
        # 이 경로의 'java' 및 'javac'가 사용됩니다.
        CODEQL_JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
        
        # CodeQL Java 익스트랙터의 루트 경로를 설정합니다.
        # 여기에는 'autobuild-fat.jar'와 같은 도구들이 있습니다.
        CODEQL_EXTRACTOR_JAVA_ROOT = "/usr/local/bin/codeql/java"
        
        # JAVA_HOME 환경 변수를 명시적으로 설정하여 빌드 스크립트 내에서 올바른 JDK를 참조하도록 합니다.
        # 이는 Jenkins tools 설정과 더불어 이중 확인 역할을 합니다.
        JAVA_HOME = "/usr/lib/jvm/java-17-amazon-corretto.x86_64"
    }

    stages {
        stage('Checkout') {
            steps {
                # Git 저장소에서 WebGoat 프로젝트를 클론합니다.
                git url: 'https://github.com/seohyun09/WebGoat.git', branch: 'develop'
            }
        }

        stage('Prepare Maven Classpath') {
            steps {
                script {
                    # WebGoat 프로젝트 디렉토리로 이동합니다.
                    dir('WebGoat') {
                        # Maven Dependency Plugin을 사용하여 모든 컴파일 종속성의 클래스패스를 생성하고
                        # 그 결과를 'classpath.txt' 파일에 저장합니다.
                        # 이 클래스패스는 'javac' 명령어가 소스 코드를 컴파일할 때 필요한 라이브러리 경로입니다.
                        sh 'mvn dependency:build-classpath -Dmdep.outputFile=classpath.txt'
                        
                        # 생성된 'classpath.txt' 파일에서 클래스패스 문자열을 읽어 Jenkins 환경 변수에 저장합니다.
                        # .trim()을 사용하여 불필요한 공백을 제거합니다.
                        env.MAVEN_CLASSPATH = readFile('classpath.txt').trim()
                        echo "Maven Classpath prepared: ${env.MAVEN_CLASSPATH}"
                    }
                }
            }
        }

        stage('CodeQL Create DB with Javac') {
            steps {
                sh '''
                    # CodeQL 데이터베이스를 생성할 프로젝트 디렉토리로 이동합니다.
                    cd WebGoat
                    
                    # 현재 사용될 JAVA_HOME과 Maven Classpath를 로그로 출력하여 확인합니다.
                    echo "Current JAVA_HOME for CodeQL: $JAVA_HOME"
                    echo "Using MAVEN_CLASSPATH for compilation: ${MAVEN_CLASSPATH}"

                    # CodeQL 데이터베이스 생성 명령을 실행합니다.
                    # --language=java: Java 언어용 데이터베이스를 생성합니다.
                    # --command="...": CodeQL에게 데이터베이스 생성을 위해 실행할 '빌드 명령'을 직접 지정합니다.
                    #                  여기서는 'javac'를 사용하여 소스 코드를 컴파일하도록 지시합니다.
                    #                  -source 17 -target 17: Java 소스 및 컴파일 버전을 17로 명시합니다.
                    #                  -cp "${MAVEN_CLASSPATH}": Maven에서 추출한 라이브러리 클래스패스를 전달합니다.
                    #                  $(find src/main/java -name "*.java"): 'src/main/java' 디렉토리의 모든 '.java' 파일을 컴파일 대상으로 지정합니다.
                    #                                                       이 부분은 프로젝트의 실제 소스 코드 위치에 따라 조정이 필요할 수 있습니다.
                    # --no-autobuild: CodeQL의 자동 빌드 로직(autobuild.sh 실행)을 명시적으로 비활성화합니다.
                    #                 이것이 핵심으로, 'release 23' 문제를 우회하는 데 중요합니다.
                    /usr/local/bin/codeql database create webgoat-db \\
                      --language=java \\
                      --command="javac -source 17 -target 17 -cp "${MAVEN_CLASSPATH}" $(find src/main/java -name "*.java")" \\
                      --no-autobuild
                '''
            }
        }

        stage('CodeQL Analyze') {
            steps {
                # CodeQL 분석을 위해 프로젝트 디렉토리로 이동합니다.
                dir('WebGoat') {
                    # CodeQL 데이터베이스를 분석합니다.
                    # 두 번째 인자는 사용할 QL 팩/스위트 경로입니다 (여기서는 Java 코드 스캔 기본 스위트).
                    # --format: 결과 포맷을 SARIF로 지정합니다.
                    # --output: SARIF 결과를 저장할 파일 경로를 지정합니다.
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
            cleanWs() // 빌드 후 워크스페이스를 정리합니다.
            echo "Workspace cleaned."
        }
    }
}
