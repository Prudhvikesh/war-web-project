pipeline {
    agent any

    tools {
        maven "maven"
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "3.109.133.197:8081"
        NEXUS_REPOSITORY = "demo-release"
        NEXUS_CREDENTIAL_ID = "nexus_credentials"
        ARTVERSION = "1.0.0"
        TOMCAT_URL = "http://43.204.147.153:8080"
        TOMCAT_CREDENTIAL_ID = "tomcat_credentials"
        TOMCAT_USERNAME = "tomcat-user"
        TOMCAT_PASSWORD = "secure-password" // Update this to the correct password
    }

    stages {
        stage('Build WAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war'
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
                    if (!fileExists(warFile)) {
                        error("WAR file not found at ${warFile}")
                    }
                    nexusArtifactUploader(
                        nexusVersion: "${NEXUS_VERSION}",
                        protocol: "${NEXUS_PROTOCOL}",
                        nexusUrl: "${NEXUS_URL}",
                        groupId: "com.example.warwebproject",
                        version: "${ARTVERSION}",
                        repository: "${NEXUS_REPOSITORY}",
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: "wwp", classifier: '', file: warFile, type: "war"],
                            [artifactId: "wwp", classifier: '', file: "pom.xml", type: "pom"]
                        ]
                    )
                }
            }
        }

stage('Deploy to Tomcat') {
    steps {
        script {
            // Find the WAR file in the target directory and use absolute path
            def warFile = sh(script: 'find target -name "*.war" -print -quit', returnStdout: true).trim()
            
            // Check if the WAR file exists, otherwise throw an error
            if (!fileExists(warFile)) {
                error("WAR file not found at ${warFile}")
            }

            // Print WAR file path to ensure it exists
            echo "WAR file located at: ${warFile}"

            // Get absolute path of the WAR file
            def absWarFile = sh(script: "realpath ${warFile}", returnStdout: true).trim()

            // Use credentials for SSH connection (passwordless authentication handled)
            sh """
                # SSH into the Tomcat server and deploy the new WAR file
                ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/jenkins_key ubuntu@43.204.147.153 <<EOF
                    # Temporarily adjust permissions for the webapps directory
                    sudo chmod -R 777 /opt/tomcat/webapps/

                    # Undeploy the existing application (if any)
                    echo "Undeploying previous version..."
                    curl -u ${TOMCAT_USERNAME}:${TOMCAT_PASSWORD} ${TOMCAT_URL}/manager/text/undeploy?path=/wwp || echo "No existing application to undeploy."

                    # Copy the new WAR file to the Tomcat webapps directory
                    echo "Deploying new WAR file..."
                    scp -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/jenkins_key ${absWarFile} ubuntu@43.204.147.153:/opt/tomcat/webapps/wwp.war

                    # Restart Tomcat to apply the changes
                    echo "Restarting Tomcat..."
                    sudo systemctl restart tomcat

                    # Restore permissions for security
                    sudo chmod -R 755 /opt/tomcat/webapps/

                    echo "Deployment complete."
                EOF
            """
        }
    }
}






    }

    post {
        always {
            echo 'Pipeline completed.'
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed. Check logs for details.'
        }
    }
}
