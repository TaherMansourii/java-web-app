pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timestamps()
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('e22f124e-2767-44cf-930d-23dd19c842a7')
        GITHUB_CREDENTIALS = credentials('da270b62-31a2-44ed-a570-c701a933abf6')
        SONAR_TOKEN = credentials('0a2bd260-f4d7-4b64-952b-b00c00f5a92b') // Sonar token
        DOCKER_IMAGE = '66raven99/java-web-app:latest'
        K8S_NAMESPACE = 'default'
        K8S_DEPLOYMENT = 'java-web-app'
        SONAR_HOST_URL = 'http://localhost:9000'
    }

    stages {

        stage('Gitleaks Secret Scan') {
            steps {
                script {
                    echo "Running Gitleaks scan..."
                    sh '''
                        if command -v gitleaks >/dev/null 2>&1; then
                            gitleaks detect --source . --verbose --redact
                        else
                            echo "Gitleaks not found! Skipping secret scan."
                        fi
                    '''
                }
            }
        }

        stage('Build Maven Project') {
            steps {
                sh './mvnw clean install'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $DOCKER_IMAGE ."
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                script {
                    echo "Scanning Docker image with Trivy..."
                    sh '''
                        if command -v trivy >/dev/null 2>&1; then
                            trivy image --exit-code 1 --severity HIGH,CRITICAL $DOCKER_IMAGE || echo "High/Critical vulnerabilities detected"
                        else
                            echo "Trivy not found! Skipping Docker image scan."
                        fi
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    echo "Running SonarQube scan..."
                    sh '''
                        if command -v mvn >/dev/null 2>&1; then
                            ./mvnw sonar:sonar \
                                -Dsonar.projectKey=java-web-app \
                                -Dsonar.host.url=$SONAR_HOST_URL \
                                -Dsonar.login=$SONAR_TOKEN
                        else
                            echo "Maven not found! Skipping SonarQube analysis."
                        fi
                    '''
                }
            }
        }

        stage('Login to DockerHub') {
            steps {
                sh "echo $DOCKERHUB_CREDENTIALS_PSW | docker login --username $DOCKERHUB_CREDENTIALS_USR --password-stdin"
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push $DOCKER_IMAGE"
            }
        }

        // Kubernetes deployment can be added later
        // stage('Deploy to Kubernetes') { ... }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs.'
        }
    }
}
