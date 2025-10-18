pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'jmp3',
                    url: 'https://github.com/Suprith25/Jenkins-mini-project.git'
            }
        }

        stage('Build') {
            steps {
                dir('sample-app') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Upload to JFrog') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'jfrog-creds',
                                                 usernameVariable: 'JFROG_USER',
                                                 passwordVariable: 'JFROG_PASS')]) {
                    sh '''
                        echo "Uploading WAR to JFrog..."
                        WAR_FILE=$(ls sample-app/target/*.war)
                        curl -u $JFROG_USER:$JFROG_PASS -T $WAR_FILE \
                        "https://trial9krpxa.jfrog.io/artifactory/testrepo-generic-local/${JOB_NAME}-${BUILD_NUMBER}-sample.war"
                    '''
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                sshagent (credentials: ['tomcat-ssh-key']) {
                    sh '''
                        echo "Deploying WAR to Tomcat server..."

                        WAR_FILE=$(ls sample-app/target/*.war)
                        SERVER_IP=172.31.7.137
                        SERVER_USER=ubuntu
                        TOMCAT_DIR=/opt/tomcat/webapps

                        # Copy WAR file to /tmp first (where ubuntu has access)
                        scp -o StrictHostKeyChecking=no $WAR_FILE $SERVER_USER@$SERVER_IP:/tmp/

                        # Move WAR into Tomcat webapps with sudo
                        ssh -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "sudo mv /tmp/$(basename $WAR_FILE) $TOMCAT_DIR/"

                        # Restart Tomcat service
                        ssh -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP "sudo systemctl restart tomcat"

                        echo "Deployment completed successfully!"
                    '''
                }
            }
        }
    }
}

