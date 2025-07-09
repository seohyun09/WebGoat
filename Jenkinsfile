pipeline {
    agent any

    stages {
        stage('📦 Checkout') {
            steps {
                checkout scm
            }
        }

        stage('🔨 Build & Test') {
            steps {
                sh './gradlew clean build'
            }
        }
    }

    post {
        always {
            cleanWs()
            echo "🧹 작업 공간 정리 완료"
        }
        success {
            echo "✅ CI 성공!"
        }
        failure {
            echo "❌ CI 실패. 로그를 확인하세요."
        }
    }
}
