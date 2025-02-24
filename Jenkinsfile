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
                    sh '''
                    if ! docker ps --format "{{.Names}}" | grep -q "sonarqube"; then
                        echo "🚀 SonarQube is not running. Starting it now..."
                        docker start sonarqube || \
                        docker run -d --name sonarqube --restart always -p 9000:9000 sonarqube:lts
                        sleep 30  # Wait for SonarQube to start
                    else
                        echo "✅ SonarQube is already running."
                    fi
                    '''
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
                    sh '''
                    echo "Removing old Docker images..." | tee -a $OUTPUT_LOG
                    docker rmi $(docker images | grep ${DOCKER_IMAGE} | awk "{print \$3}") || echo "No old images found." | tee -a $OUTPUT_LOG

                    echo "Building Docker image..." | tee -a $OUTPUT_LOG
                    docker build --no-cache -t ${DOCKER_IMAGE}:latest . | tee -a $OUTPUT_LOG || echo "Docker build failed!" | tee -a $OUTPUT_LOG

                    echo "Logging into Docker Hub..." | tee -a $OUTPUT_LOG
                    echo "$DOCKER_HUB_TOKEN" | docker login -u "$DOCKER_HUB_USERNAME" --password-stdin

                    echo "Tagging and pushing Docker image..." | tee -a $OUTPUT_LOG
                    docker tag ${DOCKER_IMAGE}:latest $DOCKER_HUB_USERNAME/${DOCKER_IMAGE}:latest
                    docker push $DOCKER_HUB_USERNAME/${DOCKER_IMAGE}:latest | tee -a $OUTPUT_LOG

                    echo "Deploying application..." | tee -a $OUTPUT_LOG
                    if [ -f docker-compose.yml ]; then
                        docker-compose pull || echo "Failed to pull latest image" | tee -a $OUTPUT_LOG
                        docker-compose down || echo "Failed to stop running containers" | tee -a $OUTPUT_LOG
                        docker-compose up -d --force-recreate --no-deps || echo "Failed to start containers" | tee -a $OUTPUT_LOG
                    else
                        echo "⚠️ No docker-compose.yml found!" | tee -a $OUTPUT_LOG
                    fi
                    '''
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    sh '''
                    echo "Running application tests..." | tee -a $OUTPUT_LOG
                    if [ -f tests/run-tests.sh ]; then
                        chmod +x tests/run-tests.sh
                        ./tests/run-tests.sh | tee -a $OUTPUT_LOG
                    else
                        echo "⚠️ No test script found!" | tee -a $OUTPUT_LOG
                    fi
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
            echo "✅ Deployment Successful!"
            slackSend(channel: '#jenkins', color: 'good', message: "✅ Deployment Successful! Build: ${env.BUILD_NUMBER} - ${env.JOB_NAME}")
        }
        failure {
            echo "❌ Deployment Failed. Check logs!"
            slackSend(channel: '#jenkins', color: 'danger', message: "❌ Deployment Failed! Build: ${env.BUILD_NUMBER} - ${env.JOB_NAME}. Check logs.")
        }
    }
}
