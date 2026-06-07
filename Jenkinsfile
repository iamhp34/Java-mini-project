pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Target Source Code Git Branch')
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/iamhp34/Java-mini-project.git', description: 'Your Personal GitHub Repository URL')
        string(name: 'APP_DIRECTORY', defaultValue: 'sample-app', description: 'Subfolder containing target Java application source')
        
        string(name: 'NEXUS_IP_PORT', defaultValue: '172.31.7.200:8081', description: 'Nexus Host IP and Port (Do not include http://)')
        string(name: 'NEXUS_REPOSITORY', defaultValue: 'maven-releases', description: 'Pre-built default hosted Nexus repository name')
        string(name: 'MAVEN_GROUP_ID', defaultValue: 'com/example', description: 'Maven Artifact Project Group ID (Use slashes / for paths)')
        
        string(name: 'DEPLOY_EC2_IP', defaultValue: '13.234.29.22', description: 'Your verified target staging EC2 instance IP address')
        string(name: 'DEPLOY_EC2_USER', defaultValue: 'ubuntu', description: 'SSH OS administrator username for your target EC2 machine')
        string(name: 'APP_DEPLOY_DIR', defaultValue: '/opt/sample-app', description: 'Standard isolated enterprise directory path on the EC2 machine')
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SONAR_SERVER_ENV = 'sonar-server'  
        NEXUS_CREDS_ID   = 'nexus-creds'    
        EC2_SSH_KEY_ID   = 'deploy-ssh-key' 
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "Stage 1: Fetching source code from branch: ${params.GIT_BRANCH}..."
                git branch: "${params.GIT_BRANCH}", url: "${params.GIT_REPO_URL}"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir("${params.APP_DIRECTORY}") {
                    echo "Stage 2: Initiating Static Code Analysis..."
                    withSonarQubeEnv("${env.SONAR_SERVER_ENV}") {
                        sh 'mvn sonar:sonar'
                    }
                }
                timeout(time: 10, unit: 'MINUTES') { 
                    echo "Evaluating default 'Sonar way' Quality Gate rules..."
                    waitForQualityGate abortPipeline: true 
                }
            }
        }

        stage('Build Stage') {
            steps {
                dir("${params.APP_DIRECTORY}") {
                    echo "Stage 3: Compiling source files and packaging application binary..."
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Push to Nexus Repo') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${env.NEXUS_CREDS_ID}",
                                                 usernameVariable: 'NEXUS_USER',
                                                 passwordVariable: 'NEXUS_PASS')]) {
                    script {
                        echo "Stage 4: Uploading compiled binary to Nexus using raw curl..."
                        
                        def localWarPath = sh(script: "ls sample-app/target/*.war", returnStdout: true).trim()
                        
                        sh """
                            curl -f -u \$NEXUS_USER:\$NEXUS_PASS --upload-file ${localWarPath} \
                            "http://${params.NEXUS_IP}:8081/repository/maven-releases/com/example/${JOB_NAME}/${BUILD_NUMBER}/${JOB_NAME}-${BUILD_NUMBER}.war"
                        """
                    }
                }
            }
        }


        stage('Deploy on EC2 Machine') {
            steps {
                sshagent(credentials: ["${env.EC2_SSH_KEY_ID}"]) {
                    withCredentials([usernamePassword(credentialsId: "${env.NEXUS_CREDS_ID}",
                                                     usernameVariable: 'NEXUS_USER',
                                                     passwordVariable: 'NEXUS_PASS')]) {
                        script {
                            def artifactName = "${JOB_NAME}"
                            def artifactVersion = "${BUILD_NUMBER}"
                            def nexusDownloadUrl = "http://${params.NEXUS_IP_PORT}/repository/${params.NEXUS_REPOSITORY}/${params.MAVEN_GROUP_ID}/${artifactName}/${artifactVersion}/${artifactName}-${artifactVersion}.war"
                            
                            echo "Stage 5: Logging into verified remote machine ${params.DEPLOY_EC2_IP}..."

                            sh """
                                ssh -o StrictHostKeyChecking=no ${params.DEPLOY_EC2_USER}@${params.DEPLOY_EC2_IP} "
                                    echo 'Setting up deployment directories under /opt...' && \
                                    sudo mkdir -p ${params.APP_DEPLOY_DIR} && \
                                    sudo chown -R ${params.DEPLOY_EC2_USER}:${params.DEPLOY_EC2_USER} ${params.APP_DEPLOY_DIR} && \
                                    
                                    echo 'Pulling isolated artifact package down directly from Nexus server...' && \
                                    curl -f -u ${NEXUS_USER}:${NEXUS_PASS} -o ${params.APP_DEPLOY_DIR}/${artifactName}.war '${nexusDownloadUrl}' && \
                                    
                                    echo 'Staging execution successfully resolved.'
                                "
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Wiping build environment runner cache..."
            cleanWs()
        }
    }
}
