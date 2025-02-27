pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "final-guestbook"
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

        stage('Remove Old Docker Image') {
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
                    sh '''
                    echo "Running Ansible Playbook..." | tee -a $OUTPUT_LOG
                    ansible-playbook auto.yml | tee -a $OUTPUT_LOG
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'pipeline_output.log', fingerprint: true
        }
        success {
            echo "✅ Pipeline Completed Successfully!"
        }
        failure {
            echo "❌ Pipeline Failed. Check logs!"
        }
    }
}
