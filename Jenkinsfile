pipeline {
    agent any

    environment {
        TOMCAT_SERVER = "13.234.34.220"
        TOMCAT_USER = "ubuntu"
        NEXUS_URL = "3.6.90.187:8081"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus_creds"
        SSH_KEY_PATH = "/var/lib/jenkins/.ssh/jenkins_key"
        SONAR_HOST_URL = "http://13.235.243.105:9000"
        SONAR_CREDENTIAL_ID = "sonar_creds"
    }

    tools {
        maven "maven" // Make sure this matches your Jenkins global tool config
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build WAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    withCredentials([string(credentialsId: env.SONAR_CREDENTIAL_ID, variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn sonar:sonar \
                                -Dsonar.projectKey=wwp \
                                -Dsonar.host.url=${env.SONAR_HOST_URL} \
                                -Dsonar.login=${SONAR_TOKEN} \
                                -Dsonar.java.binaries=target/classes
                        """
                    }
                }
            }
        }

        stage('Extract Version') {
            steps {
                script {
                    env.ART_VERSION = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    echo "🔖 Version extracted: ${env.ART_VERSION}"
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    echo "📤 Uploading ${warFile} to Nexus..."
                    nexusArtifactUploader(
                        nexusVersion: "nexus3",
                        protocol: "http",
                        nexusUrl: "${env.NEXUS_URL}",
                        groupId: "koddas.web.war",
                        version: "${env.ART_VERSION}",
                        repository: "${env.NEXUS_REPOSITORY}",
                        credentialsId: "${env.NEXUS_CREDENTIAL_ID}",
                        artifacts: [[
                            artifactId: "wwp",
                            classifier: '',
                            file: warFile,
                            type: "war"
                        ]]
                    )
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    echo "🚀 Deploying ${warFile} to Tomcat at ${env.TOMCAT_SERVER}..."
                    sh """
                        scp -i ${env.SSH_KEY_PATH} -o StrictHostKeyChecking=no ${warFile} ${env.TOMCAT_USER}@${env.TOMCAT_SERVER}:/tmp/
                        ssh -i ${env.SSH_KEY_PATH} -o StrictHostKeyChecking=no ${env.TOMCAT_USER}@${env.TOMCAT_SERVER} '
                            sudo mv /tmp/*.war /opt/tomcat/webapps/ && sudo systemctl restart tomcat'
                    """
                }
            }
        }

        stage('Display URLs') {
            steps {
                script {
                    def appUrl = "http://${env.TOMCAT_SERVER}:8080/wwp-${env.ART_VERSION}/"
                    def nexusUrl = "http://${env.NEXUS_URL}/repository/${env.NEXUS_REPOSITORY}/koddas/web/war/wwp/${env.ART_VERSION}/wwp-${env.ART_VERSION}.war"

                    echo "🌐 App URL: ${appUrl}"
                    echo "📦 Nexus URL: ${nexusUrl}"
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Please check logs.'
        }
    }
}
