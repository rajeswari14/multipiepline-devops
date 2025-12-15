pipeline {
    agent any

    environment {
        APP_NAME    = "springboot-app"
        APP_DIR     = "/opt/${APP_NAME}"
        DEPLOY_USER = "ubuntu"
        DEPLOY_HOST = "44.203.106.7"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package JAR') {
            steps {
                sh '''
                    mvn clean package -DskipTests
                    ls -lh target/*.jar
                '''
            }
        }

        stage('Security Scan (Trivy)') {
            steps {
                sh '''
                    trivy fs --exit-code 0 --format json \
                    --output trivy-report.json .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
                }
            }
        }

        stage('Deploy to App EC2 (main only)') {
            when {
                branch 'main'
            }
            steps {
                sshagent(['asst-1']) {

                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@44.203.106.7 "mkdir -p /opt/${APP_NAME}"
                        scp -o StrictHostKeyChecking=no target/*.jar ubuntu@44.203.106.7:/opt/${APP_NAME}/app.jar
                        ssh -o StrictHostKeyChecking=no ubuntu@44.203.106.7 "pkill -f app.jar || true"
                        "nohup java -jar /opt/${APP_NAME}/app.jar > /opt/${APP_NAME}/app.log 2>&1 &"
                    '''
                }
            }
        }

    }

    post {
        success {
            echo "SUCCESS on branch ${env.BRANCH_NAME}"
        }
        failure {
            echo "FAILED on branch ${env.BRANCH_NAME}"
        }
    }
}
