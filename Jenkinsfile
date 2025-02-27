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
        stage('Run Ansible to Ensure Jenkins & Cleanup') {
            steps {
                script {
                    sh '''
                    echo "Running Ansible Playbook to Ensure Jenkins & Remove Old Images..." | tee -a $OUTPUT_LOG
                    ansible-playbook auto.yml | tee -a $OUTPUT_LOG
                    '''
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

        stage('Run Tests') {
            steps {
                script {
                    sh '''
                    echo "Running Unit Tests..." | tee -a $OUTPUT_LOG
                    ./run_tests.sh | tee -a $OUTPUT_LOG
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                        echo "Running SonarQube Analysis..." | tee -a $OUTPUT_LOG
                        sonar-scanner \
                            -Dsonar.projectKey=${SONARQUBE_PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${SONAR_HOST_URL} | tee -a $OUTPUT_LOG
                        '''
                    }
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

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker-hub-token', variable: 'DOCKER_HUB_TOKEN')]) {
                        sh '''
                        echo "$DOCKER_HUB_TOKEN" | docker login -u "$DOCKER_HUB_USERNAME" --password-stdin
                        docker tag ${DOCKER_IMAGE}:latest $DOCKER_HUB_USERNAME/${DOCKER_IMAGE}:latest
                        docker push $DOCKER_HUB_USERNAME/${DOCKER_IMAGE}:latest | tee -a $OUTPUT_LOG
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Saving Pipeline Output Log..."
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
