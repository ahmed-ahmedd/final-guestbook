pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "final-guestbook"
        SONARQUBE_PROJECT_KEY = "final-guestbook"
        SONAR_HOST_URL = "http://16.170.182.27:9000"
        DOCKER_HUB_USERNAME = "ahmedelshandidy"
        OUTPUT_LOG = "pipeline_output.log"
    }

    stages {
        stage('Cleanup') {
            steps {
                script {
                    sh 'echo "Starting Cleanup..." > $OUTPUT_LOG'
                    sh 'git clean -fdx'
                    sh 'rm -f $OUTPUT_LOG || true'
                }
            }
        }

        stage('Checkout Code') {
            steps {
                script {
                    checkout scm
                    sh 'ls -la | tee -a $OUTPUT_LOG'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=${SONARQUBE_PROJECT_KEY} \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=${SONAR_HOST_URL} \
                          -Dsonar.login=$SONAR_TOKEN \
                          -Dsonar.qualitygate.wait=true \
                          -Dsonar.exclusions="**/node_modules/**,**/tests/**,**/*.log,**/bin/**,**/out/**" \
                          | tee -a $OUTPUT_LOG
                        '''
                    }
                }
            }
        }

        stage('Remove Old Docker Images') {
            steps {
                script {
                    sh '''
                    echo "Removing old Docker images..." | tee -a $OUTPUT_LOG
                    docker rmi $(docker images -q ${DOCKER_IMAGE}) || echo "No old images found." | tee -a $OUTPUT_LOG
                    '''
                }
            }
        }

        stage('Build Docker Image (No Cache)') {
            steps {
                script {
                    sh '''
                    echo "Building Docker image..." | tee -a $OUTPUT_LOG
                    docker build --no-cache -t ${DOCKER_IMAGE}:latest . | tee -a $OUTPUT_LOG
                    '''
                }
            }
        }

        stage('Run Ansible Playbook') {
            steps {
                script {
                    sh 'ansible-playbook auto.yml | tee -a $OUTPUT_LOG'
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'pipeline_output.log', fingerprint: true
        }
        success {
            echo "✅ Deployment Successful!"
            slackSend(channel: '#jenkins', color: 'good', message: "✅ Deployment Successful! Build: ${env.BUILD_NUMBER} - ${env.JOB_NAME}")
        }
        failure {
            echo "❌ Deployment Failed. Check logs!"
            slackSend(channel: '#jenkins', color: 'danger', message: "❌ Deployment Failed! Build: ${env.BUILD_NUMBER} - ${env.JOB_NAME}. Check logs.")
        }
    }
}
