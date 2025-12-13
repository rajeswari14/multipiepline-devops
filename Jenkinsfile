pipeline {
    agent { label 'agent' }

    environment {
        APP_NAME    = "springboot-app"
        APP_DIR     = "/opt/${APP_NAME}"
        DEPLOY_USER = "ubuntu"
        DEPLOY_HOST = "44.193.0.46"

        JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
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
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package JAR') {
            steps {
                sh '''
                    mvn clean package -DskipTests
                    echo "📦 Packaged Artifacts:"
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
                sshagent(['app-server']) {
                    sh '''
                        echo "🚀 Deploying to Application EC2..."

                        # 1️⃣ Ensure app directory exists
                        ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_HOST} \
                          "mkdir -p ${APP_DIR}"

                        # 2️⃣ Copy JAR to App EC2
                        scp -o StrictHostKeyChecking=no \
                          target/*.jar \
                          ${DEPLOY_USER}@${DEPLOY_HOST}:${APP_DIR}/app.jar

                        # 3️⃣ Stop old app & start new one
                        ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_HOST} "
                            pkill -f app.jar || true
                            nohup java -jar ${APP_DIR}/app.jar \
                              > ${APP_DIR}/app.log 2>&1 &
                        "

                        echo "✅ Deployment Successful"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "🎉 PIPELINE SUCCESS on branch ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ PIPELINE FAILED on branch ${env.BRANCH_NAME}"
        }
    }
}
