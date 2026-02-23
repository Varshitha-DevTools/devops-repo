pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17'
            args '-v /var/run/docker.sock:/var/run/docker.sock -u root'
        }
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        SLACK_CHANNEL = "#all-devtools"   // Change to your Slack channel
    }

    stages {
        stage('Verify Environment') {
            steps {
                sh 'java -version'
                sh 'mvn -version'
                sh 'docker --version'
            }
        }

        stage("Build Application") {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        def imagePath = "${DOCKER_USER}/${APP_NAME}"
                        def fullImage = "${imagePath}:${IMAGE_TAG}"
                        
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        sh "docker build -t ${fullImage} ."
                        sh "docker tag ${fullImage} ${imagePath}:latest"
                        sh "docker push ${fullImage}"
                        sh "docker push ${imagePath}:latest"
                        sh "docker logout"
                    }
                }
            }
        }

        stage("Trivy Vulnerability Scan") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'UNUSED_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh """
                            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy image ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG} \
                            --severity HIGH,CRITICAL --format table
                        """
                    }
                }
            }
        }

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }
    }

    // ✅ Slack Notifications Section
    post {
        success {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: "good",
                message: """
✅ *BUILD SUCCESS*
Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
Image: ${APP_NAME}:${IMAGE_TAG}
URL: ${env.BUILD_URL}
"""
            )
        }

        failure {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: "danger",
                message: """
❌ *BUILD FAILED*
Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
Check logs: ${env.BUILD_URL}
"""
            )
        }

        unstable {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: "warning",
                message: """
⚠️ *BUILD UNSTABLE*
Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
URL: ${env.BUILD_URL}
"""
            )
        }
    }
}
