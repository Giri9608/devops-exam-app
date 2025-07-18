pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "giri8608/devopsexamapp:latest"
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/Giri9608/devops-exam-app.git', 
                    branch: 'master'
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --security-checks vuln,config --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=devops-exam-app \
                    -Dsonar.projectKey=devops-exam-app \
                    -Dsonar.sources=. \
                    -Dsonar.language=py \
                    -Dsonar.python.version=3 \
                    -Dsonar.host.url=http://localhost:9000
                    """
                }
            }
        }

        stage('Verify Docker Compose') {
            steps {
                sh '''
                docker compose version || { echo "Docker Compose not available"; exit 1; }
                '''
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                dir('backend') {
                    script {
                        withDockerRegistry(credentialsId: 'docker-creds', toolName: 'docker') {
                            sh "docker build -t ${DOCKER_IMAGE} ."
                            sh "docker push ${DOCKER_IMAGE}"
                        }
                    }
                }
            }
        }

        stage('Docker Scout Image Analysis') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker scout quickview ${DOCKER_IMAGE}"
                        sh "docker scout cves ${DOCKER_IMAGE}"
                        sh "docker scout recommendations ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                # Clean up any existing containers
                docker compose down --remove-orphans || true
                
                # Start services
                docker compose up -d
                
                # Wait for MySQL to be ready
                echo "Waiting for MySQL to be ready..."
                timeout 120s bash -c '
                while ! docker compose exec -T mysql mysqladmin ping -uroot -prootpass --silent;
                do 
                    sleep 5;
                    docker compose logs mysql --tail=5 || true;
                done'
                
                # Additional wait for full initialization
                sleep 10
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                echo "=== Container Status ==="
                docker compose ps -a
                echo "=== Testing Flask Endpoint ==="
                curl -I http://localhost:5000 || true
                '''
            }
        }
    }

    post {
        success {
            echo '🚀 Deployment successful!'
            sh 'docker compose ps'
            sh 'docker images | grep devopsexamapp'
            archiveArtifacts artifacts: 'trivy-fs-report.html', allowEmptyArchive: true
        }
        failure {
            echo '❗ Pipeline failed. Check logs above.'
            sh '''
            echo "=== Error Investigation ==="
            docker compose logs --tail=50 || true
            '''
            archiveArtifacts artifacts: 'trivy-fs-report.html', allowEmptyArchive: true
        }
        always {
            sh '''
            echo "=== Final Logs ==="
            docker compose logs --tail=20 || true
            '''
            archiveArtifacts artifacts: 'trivy-fs-report.html', allowEmptyArchive: true
        }
    }
}
