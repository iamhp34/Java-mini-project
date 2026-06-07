pipeline {
    agent any

    // Dynamic abstraction inputs ensuring zero hardcoding across your real environment
    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Target Source Code Git Branch')
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/iamhp34/Java-mini-project.git', description: 'Your Personal GitHub Repository URL')
        string(name: 'APP_DIRECTORY', defaultValue: 'sample-app', description: 'Subfolder containing target Java application source')
        
        // Nexus Storage Environment Parameters
        string(name: 'NEXUS_IP_PORT', defaultValue: '172.31.7.200:8081', description: 'Nexus Host IP and Port (Do not include http://)')
        string(name: 'NEXUS_REPOSITORY', defaultValue: 'maven-releases', description: 'Pre-built default hosted Nexus repository name')
        string(name: 'MAVEN_GROUP_ID', defaultValue: 'com/example', description: 'Maven Artifact Project Group ID (Use slashes / for paths)')
        
        // Your Pre-Ready Deployment EC2 Node Details
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
        EC2_SSH_KEY_ID   = 'tomcat-ssh-key' // Maps the secure .pem key you used to establish your successful SSH connection
    }

    stages {
        // STAGE 1: Code Checkout
        stage('Checkout Code') {
            steps {
                echo "Stage 1: Fetching source code from branch: ${params.GIT_BRANCH}..."
                git branch: "${params.GIT_BRANCH}", url: "${params.GIT_REPO_URL}"
            }
        }

        // STAGE 2: SonarQube Quality Analysis
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

        // STAGE 3: Build & Test
        stage('Build Stage') {
            steps {
                dir("${params.APP_DIRECTORY}") {
                    echo "Stage 3: Compiling source files and packaging application binary..."
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        // STAGE 4: Push to Nexus Repo
        stage('Push to Nexus Repo') {
            steps {
                script {
                    def warFile = findFiles(glob: "${params.APP_DIRECTORY}/target/*.war")
                    if (!warFile) { error "No compiled artifact (.war) found for Nexus upload!" }
                    
                    def foundWarPath = warFile.path
                    echo "Stage 4: Uploading ${foundWarPath} to Nexus Open-Source Server..."
                    
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${params.NEXUS_IP_PORT}",
                        groupId: "${params.MAVEN_GROUP_ID}".replaceAll('/', '.'), 
                        version: "${BUILD_NUMBER}",
                        repository: "${params.NEXUS_REPOSITORY}",
                        credentialsId: "${env.NEXUS_CREDS_ID}",
                        artifacts: [
                            [artifactId: "${JOB_NAME}", file: "${foundWarPath}", type: 'war']
                        ]
                    )
                }
            }
        }

        // STAGE 5: Deploy on EC2 Machine (Fetching artifact from Nexus only)
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
