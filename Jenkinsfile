pipeline {
    agent any

    tools {
        jdk 'JDK_17'
        maven 'M3'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/seohyun09/WebGoat.git', branch: 'develop'
            }
        }

        stage('Build & FindSecBugs') {
            steps {
                // FindSecBugs 플러그인 포함 빌드 및 분석
                sh 'mvn clean package com.h3xstream.findsecbugs:findsecbugs-maven-plugin:1.12.0:findsecbugs'
            }
        }

        stage('Publish FindSecBugs Report') {
            steps {
                // Jenkins Warnings Next Generation 플러그인을 사용하는 예시
                recordIssues tools: [spotBugs(pattern: '**/target/findbugsXml.xml')]
                // 또는 HTML 리포트라면
                // publishHTML([reportDir: 'target/site', reportFiles: 'findsecbugs.html', reportName: 'FindSecBugs Report'])
            }
        }
    }

    post {
        success {
            echo "✅ FindSecBugs 정적 분석 및 리포트 게시 완료!"
        }
        failure {
            echo "❌ FindSecBugs 분석 또는 리포트 게시에 실패했습니다. 로그를 확인하세요."
        }
        always {
            cleanWs()
            echo "Workspace cleaned."
        }
    }
}
