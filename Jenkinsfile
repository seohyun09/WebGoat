pipeline {
    agent { label 'sast-infer-agent' } // 이 agent가 Maven, JDK, Git, Docker를 모두 가지고 있어야 합니다.

    tools {
        jdk 'JDK_17' // Jenkins Global Tool Configuration에 'JDK_17'로 JDK 17이 설정되어 있어야 합니다.
        maven 'M3'   // Jenkins Global Tool Configuration에 'M3'와 같은 이름으로 Maven이 설정되어 있어야 합니다.
                     // (혹은 pipeline 내에서 sh 'mvn ...' 명령 시 PATH에 Maven이 잡혀있어야 합니다.)
    }

    environment {
        // 기존 AWS 관련 환경 변수들은 주석 처리 또는 제거되었습니다.
        // 필요하다면 다시 활성화하거나 추가하세요.
    }

    stages {
        stage('Checkout') {
            steps {
                // 'credentialsId: 'none''은 퍼블릭 저장소에 적합합니다.
                // 만약 프라이빗 저장소라면 적절한 credentialsId를 사용해야 합니다.
                git url: 'https://github.com/WebGoat/WebGoat.git', credentialsId: 'none'
            }
        }

        stage('Static Analysis with Infer') {
            steps {
                script {
                    def workspacePath = pwd()

                    echo "Starting Infer static analysis for WebGoat..."
                    // WebGoat 프로젝트의 pom.xml이 webgoat-parent/pom.xml에 있을 수 있습니다.
                    // 실제 프로젝트 구조에 맞게 경로를 확인하고 수정해야 합니다.
                    sh "docker run --rm -v ${workspacePath}:/src facebook/infer:latest infer -- mvn -f /src/pom.xml clean install"
                    echo "Infer analysis completed. Collecting results..."

                    // Infer 결과 아티팩트 보관
                    archiveArtifacts artifacts: 'infer-out/**', fingerprint: true, allowEmpty: true

                    echo "--- Infer Analysis Report (report.json) ---"
                    sh "cat infer-out/report.json || echo 'infer-out/report.json not found or empty.'"
                    echo "------------------------------------------"
                }
            }
        }

        // --- FindSecBugs 분석 스테이지 추가 ---
        stage('Static Analysis with FindSecBugs') {
            steps {
                script {
                    echo "Starting FindSecBugs analysis for WebGoat..."
                    // pom.xml에 spotbugs-maven-plugin이 설정되어 있다고 가정합니다.
                    // 'M3'는 Jenkins Global Tool Configuration에 설정된 Maven 이름입니다.
                    // 프로젝트의 기본 pom.xml을 사용하여 spotbugs를 실행합니다.
                    withMaven(maven: 'M3') {
                        sh 'mvn clean install spotbugs:spotbugs'
                    }
                    echo "FindSecBugs analysis completed. Publishing results..."
                }
            }
            post {
                always {
                    // Warnings Next Generation Plugin을 사용하여 SpotBugs/FindSecBugs 결과를 게시합니다.
                    // pom.xml에서 지정한 출력 파일 경로와 일치해야 합니다.
                    recordIssues enabledForFailure: true, tools: [
                        spotBugs(pattern: '**/target/spotbugsXml.xml')
                    ]
                }
            }
        }
        // --- FindSecBugs 분석 스테이지 추가 끝 ---

        // 기존에 주석 처리했던 Docker 및 AWS 관련 스테이지는 그대로 주석 처리하거나 필요에 따라 제거합니다.
        // stage('Build Docker Image') { /* ... */ }
        // stage('Login to AWS ECR') { /* ... */ }
        // stage('Push to ECR') { /* ... */ }
        // stage('Deploy via CodeDeploy') { /* ... */ }
    }

    post {
        success {
            echo "✅ All static analyses completed successfully!"
        }
        failure {
            echo "❌ Some static analysis failed. Please check logs."
        }
        always {
            // 빌드가 끝난 후 워크스페이스를 정리합니다.
            cleanWs()
            echo "Cleanup complete."
        }
    }
}
