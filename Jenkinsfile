pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('e22f124e-2767-44cf-930d-23dd19c842a7')
        GITHUB_CREDENTIALS = credentials('da270b62-31a2-44ed-a570-c701a933abf6')
        SONAR_TOKEN = credentials('0a2bd260-f4d7-4b64-952b-b00c00f5a92b') // Sonar token stored in Jenkins
        DOCKER_IMAGE = '66raven99/java-web-app:latest'
        K8S_NAMESPACE = 'default'
        K8S_DEPLOYMENT = 'java-web-app'
    }

    stages {
        stage('Gitleaks Secret Scan') {
            steps {
                sh '''
                    echo "Running Gitleaks scan..."
                    gitleaks detect --source . --verbose --redact
                '''
            }
        }

        stage('Build Maven Project') {
            steps {
                sh './mvnw clean install'
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                sh """
                    echo "Scanning Docker image with Trivy..."
                    trivy image --exit-code 1 --severity HIGH,CRITICAL $DOCKER_IMAGE || true
                """
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONAR_HOST_URL = 'http://localhost:9000'
            }
            steps {
                sh """
                    echo "Running SonarQube scan..."
                    ./mvnw sonar:sonar \
                        -Dsonar.projectKey=java-web-app \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_TOKEN
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $DOCKER_IMAGE ."
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

        // You can uncomment and fix your Kubernetes deployment stage later
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
