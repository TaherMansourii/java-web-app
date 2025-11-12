pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 60, unit: 'MINUTES') // optional: prevent long-running jobs
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('e22f124e-2767-44cf-930d-23dd19c842a7')
        GITHUB_CREDENTIALS = credentials('da270b62-31a2-44ed-a570-c701a933abf6')
        SONAR_TOKEN = credentials('0a2bd260-f4d7-4b64-952b-b00c00f5a92b')
        DOCKER_IMAGE = '66raven99/java-web-app:latest'
        K8S_NAMESPACE = 'default'
        K8S_DEPLOYMENT = 'java-web-app'
        SONAR_HOST_URL = 'http://192.168.33.10:9000'
    }

    stages {
        stage('Gitleaks Secret Scan') {
            steps {
                sh '''
                    echo "Running Gitleaks scan..."
                    gitleaks detect --source . --verbose --redact || true
                '''
            }
        }

        stage('Build Maven Project') {
            steps {
                sh './mvnw clean install'
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                sh '''
                    echo "Running OWASP Dependency-Check..."
                    ./mvnw org.owasp:dependency-check-maven:check \
                        -Dformat=HTML \
                        -DoutputDirectory=dependency-check-report || true
                '''
                archiveArtifacts artifacts: 'dependency-check-report/**', allowEmptyArchive: true
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
            steps {
                withSonarQubeEnv('SonarQube') { // Name of the SonarQube installation in Jenkins
                    sh """
                        echo "Running SonarQube scan..."
                        ./mvnw sonar:sonar \
                            -Dsonar.projectKey=java-web-app \
                            -Dsonar.projectName='Java Web App' \
                            -Dsonar.host.url=$SONAR_HOST_URL \
                            -Dsonar.login=$SONAR_TOKEN
                    """
                }
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

        // Uncomment and configure this stage if you want Kubernetes deployment
        /*
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-minikube', variable: 'KUBECONFIG')]) {
                    sh """
                        echo "Deploying to Kubernetes..."
                        kubectl --kubeconfig="$KUBECONFIG" set image deployment/$K8S_DEPLOYMENT $K8S_DEPLOYMENT=$DOCKER_IMAGE -n $K8S_NAMESPACE
                        kubectl --kubeconfig="$KUBECONFIG" rollout status deployment/$K8S_DEPLOYMENT -n $K8S_NAMESPACE
                    """
                }
            }
        }
        */
        stage('Deploy with Docker Compose') {
    steps {
        sh """
            docker-compose -f docker-compose.yml pull
            docker-compose -f docker-compose.yml up -d
        """
    }
}

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
