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
        DOCKER_CREDS = "dockerhub"        // Jenkins credentials ID for Docker Hub
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
                    withSonarQubeEnv('jenkins-sonarqube-token') {
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

        stage('Check Docker Access') {
            steps {
                sh 'docker ps'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    // Build the Docker image
                    def dockerImage = docker.build("${IMAGE_NAME}", ".")

                    // Push to Docker Hub using stored credentials
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDS}") {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push("latest")
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        success {
            echo 'Build and push completed successfully!'
        }
        failure {
            echo 'Build failed. Please check logs for details.'
        }
    }
}
