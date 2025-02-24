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
                    sh 'rm -rf * || true'
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

        stage('Verify Environment') {
            steps {
                script {
                    sh '''
                    docker --version || echo "Docker not installed!" | tee -a $OUTPUT_LOG
                    docker-compose --version || echo "Docker Compose not found!" | tee -a $OUTPUT_LOG
                    sonar-scanner --version || echo "SonarScanner not installed!" | tee -a $OUTPUT_LOG
                    ansible --version || echo "Ansible not installed!" | tee -a $OUTPUT_LOG
                    '''
                }
            }
        }

        stage('Ensure SonarQube is Running') {
            steps {
                script {
                    ansiblePlaybook(
                        playbook: 'ansible/playbooks/sonarqube.yml',
                        inventory: 'ansible/inventory/hosts'
                    )
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

        stage('Deploy Application') {
            steps {
                script {
                    ansiblePlaybook(
                        playbook: 'ansible/playbooks/deploy.yml',
                        inventory: 'ansible/inventory/hosts',
                        extraVars: [
                            docker_image: "${DOCKER_IMAGE}",
                            docker_hub_username: "${DOCKER_HUB_USERNAME}"
                        ]
                    )
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    ansiblePlaybook(
                        playbook: 'ansible/playbooks/tests.yml',
                        inventory: 'ansible/inventory/hosts'
                    )
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
