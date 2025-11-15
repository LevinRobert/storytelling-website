pipeline {
    agent { label 'jenkins-agent' }

    tools {
        jdk 'java17'
        maven 'maven3'
    }

    environment {
        APP_NAME = "redefinee-website-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "levin16robert"
        DOCKER_CREDS = "dockerhub"          // Jenkins credentials ID for Docker Hub
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from SCM') {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/LevinRobert/storytelling-website.git'
                sh 'ls -lrt'
            }
        }

        stage('Build Application') {
            steps {
                sh 'pwd'
                sh 'ls'
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test Application') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('sonarqube-service') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }
        }
        stage('Debug User') {
            steps {
                sh 'whoami'
                sh 'id'
                sh 'ls -l /var/run/docker.sock'
    }
}


        stage('Docker build and push image') {
            steps {
                sh "ls -lrt"
                sh "pwd"
                
                }
            }
        }
    }
