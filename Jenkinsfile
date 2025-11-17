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
        DOCKER_CREDS = "dockerhub"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}".toLowerCase()
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("jenkins-api-token")
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
                sh "docker system prune -af || true"
            }
        }

        stage('Checkout from SCM') {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/LevinRobert/redefinee-website.git'
                sh 'ls -lrt'
            }
        }

        stage('Build Application') {
            steps {
                sh 'mvn clean package -DskipTests'
                sh 'ls -R target'
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

        stage('Debug Info') {
            steps {
                sh 'whoami'
                sh 'id'
                sh 'ls -l /var/run/docker.sock'
                sh 'ls -R .'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {

                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."

                    withCredentials([
                        usernamePassword(
                            credentialsId: "${DOCKER_CREDS}",
                            usernameVariable: 'DH_USER',
                            passwordVariable: 'DH_PASS'
                        )
                    ]) {
                        sh '''
                            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        '''
                    }

                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
                    sh "docker push ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Trivy Scan (Fix: Avoid Cache Lock Issue)') {
            steps {
                script {
                    def TRIVY_CACHE = "/tmp/trivy-cache-${BUILD_NUMBER}"

                    sh "mkdir -p ${TRIVY_CACHE}"

                    sh """
                        docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v ${TRIVY_CACHE}:/root/.cache/ \
                        aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} \
                        --no-progress --scanners vuln --exit-code 0 \
                        --severity HIGH,CRITICAL --format table
                    """

                    sh "rm -rf ${TRIVY_CACHE} || true"
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                    sh "docker system prune -af || true"
                }
            }
        }

        /* -------------------------------------------------------------- */
        /*  NEW STAGE: UPDATE GITOPS REPO FOR ARGOCD AUTO-DEPLOY         */
        /* -------------------------------------------------------------- */

        stage('Update GitOps deployment.yaml') {
            steps {
                script {

                    withCredentials([
                        usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')
                    ]) {

                        sh """
                            rm -rf gitops
                            git clone https://${GIT_USER}:${GIT_PASS}@github.com/LevinRobert/k8s-gitops.git gitops

                            cd gitops/k8s

                            # Update image tag in deployment.yaml
                            sed -i 's|image: .*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g' deployment.yaml

                            git config user.email "jenkins@ci.com"
                            git config user.name "Jenkins CI"

                            git add .
                            git commit -m "Update image tag to ${IMAGE_TAG}"
                            git push
                        """
                    }
                }
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                script {
                    build job: 'redefinee-website-CD',
                          wait: false,
                          parameters: [
                              string(name: 'IMAGE_NAME', value: IMAGE_NAME),
                              string(name: 'IMAGE_TAG', value: IMAGE_TAG)
                          ]

                    echo "Triggered CD pipeline for deployment"
                }
            }
        }
    }
}
