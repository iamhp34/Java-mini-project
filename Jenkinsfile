pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Target Source Code Git Branch')
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/iamhp34/Java-mini-project.git', description: 'Your Personal GitHub Repository URL')
        string(name: 'APP_DIRECTORY', defaultValue: 'sample-app', description: 'Subfolder containing target Java application source')

        string(name: 'NEXUS_IP_PORT', defaultValue: '172.31.7.200:8081', description: 'Nexus Host IP and Port')
        string(name: 'NEXUS_REPOSITORY', defaultValue: 'maven-releases', description: 'Nexus hosted repository name')
        string(name: 'MAVEN_GROUP_ID', defaultValue: 'com/example', description: 'Maven Artifact Group ID path')

        string(name: 'DEPLOY_EC2_IP', defaultValue: '13.234.29.22', description: 'Target staging EC2 IP')
        string(name: 'DEPLOY_EC2_USER', defaultValue: 'ubuntu', description: 'SSH username')
        string(name: 'APP_DEPLOY_DIR', defaultValue: '/opt/sample-app', description: 'Deployment directory on EC2')
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
                    echo "Evaluating SonarQube Quality Gate rules..."
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Stage') {
            steps {
                dir("${params.APP_DIRECTORY}") {
                    echo "Stage 3: Compiling and packaging application..."
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Push to Nexus Repo') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${env.NEXUS_CREDS_ID}",
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {
                    script {
                        echo "Stage 4: Uploading WAR file to Nexus..."

                        def localWarPath = sh(
                            script: "ls ${params.APP_DIRECTORY}/target/*.war",
                            returnStdout: true
                        ).trim()

                        def artifactName = "${JOB_NAME}"
                        def artifactVersion = "${BUILD_NUMBER}"

                        sh """
                            curl -f -u \$NEXUS_USER:\$NEXUS_PASS --upload-file ${localWarPath} \
                            "http://${params.NEXUS_IP_PORT}/repository/${params.NEXUS_REPOSITORY}/${params.MAVEN_GROUP_ID}/${artifactName}/${artifactVersion}/${artifactName}-${artifactVersion}.war"
                        """
                    }
                }
            }
        }

        stage('Deploy on EC2 Machine') {
            steps {
                sshagent(credentials: ["${env.EC2_SSH_KEY_ID}"]) {
                    withCredentials([
                        usernamePassword(
                            credentialsId: "${env.NEXUS_CREDS_ID}",
                            usernameVariable: 'NEXUS_USER',
                            passwordVariable: 'NEXUS_PASS'
                        )
                    ]) {
                        script {
                            def artifactName = "${JOB_NAME}"
                            def artifactVersion = "${BUILD_NUMBER}"

                            def nexusDownloadUrl = "http://${params.NEXUS_IP_PORT}/repository/${params.NEXUS_REPOSITORY}/${params.MAVEN_GROUP_ID}/${artifactName}/${artifactVersion}/${artifactName}-${artifactVersion}.war"

                            echo "Stage 5: Deploying artifact on EC2: ${params.DEPLOY_EC2_IP}..."

                            sh """
                                ssh -o StrictHostKeyChecking=no ${params.DEPLOY_EC2_USER}@${params.DEPLOY_EC2_IP} "
                                    echo 'Creating deployment directory...' && \
                                    sudo mkdir -p ${params.APP_DEPLOY_DIR} && \
                                    sudo chown -R ${params.DEPLOY_EC2_USER}:${params.DEPLOY_EC2_USER} ${params.APP_DEPLOY_DIR} && \

                                    echo 'Downloading WAR file from Nexus...' && \
                                    curl -f -u \$NEXUS_USER:\$NEXUS_PASS -o ${params.APP_DEPLOY_DIR}/${artifactName}.war '${nexusDownloadUrl}' && \

                                    echo 'Deployment completed successfully.'
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