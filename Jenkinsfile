pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-northeast-2'
        ECR_REPO = '592992781155.dkr.ecr.ap-northeast-2.amazonaws.com/devsecops_ecr'
        IMAGE_TAG = 'latest'
        SONARQUBE_ENV = 'SonarQube' // Jenkins에 등록된 SonarQube 서버 이름
        SONAR_HOST_URL = 'http://13.125.254.54:9000' // 실제 SonarQube 서버 주소
        SONAR_AUTH_TOKEN = credentials('sonarqube-token') // Jenkins에 등록된 SonarQube 토큰 ID
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=webgoat \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        stage('Fetch SonarQube Report via API') {
            steps {
                script {
                    def timestamp = sh(script: "date +%F_%H-%M-%S", returnStdout: true).trim()
                    env.REPORT_FILE = "sonar_issues_${timestamp}.json"

                    sh """
                        curl -s -H "Authorization: Bearer $SONAR_AUTH_TOKEN" \\
                             "$SONAR_HOST_URL/api/issues/search?componentKeys=webgoat&severities=CRITICAL,BLOCKER" \\
                             -o ${env.REPORT_FILE}
                    """
                }
            }
        }

        stage('Upload Sonar Report to S3') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws_credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set region $AWS_REGION

                        aws s3 cp $REPORT_FILE s3://sonia-bucket-for-sonarqube/sonarqube-reports/$REPORT_FILE
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $ECR_REPO:$IMAGE_TAG .'
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws_credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set region $AWS_REGION

                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login --username AWS --password-stdin $ECR_REPO
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh 'docker push $ECR_REPO:$IMAGE_TAG'
            }
        }
    }
}
