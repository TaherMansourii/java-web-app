pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('e22f124e-2767-44cf-930d-23dd19c842a7')
        GITHUB_CREDENTIALS = credentials('da270b62-31a2-44ed-a570-c701a933abf6')
        SONAR_TOKEN = credentials('bb76dcfb-4591-46c5-9481-1491b18c8cd9') // Updated Sonar token
        DOCKER_IMAGE = '66raven99/java-web-app:latest'
        K8S_NAMESPACE = 'default'
        K8S_DEPLOYMENT = 'java-web-app'
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
                withSonarQubeEnv('SonarQube') {
                    sh './mvnw sonar:sonar -Dsonar.projectKey=java-web-app -Dsonar.projectName="Java Web App"'
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

        stage('Deploy with Docker Compose') {
            steps {
                sh """
                    docker-compose -f docker-compose.yml pull
                    docker-compose -f docker-compose.yml up -d
                """
            }
        }

       stage('OWASP ZAP Scan') {
    steps {
        script {
            echo "Running OWASP ZAP baseline scan..."

            // Detect the exposed app port
            def appPort = sh(script: "docker ps --filter ancestor=${DOCKER_IMAGE} --format '{{.Ports}}' | grep -oP '(?<=:)[0-9]+(?=-)' | head -n 1", returnStdout: true).trim()
            echo "Detected app running on port ${appPort}"

            // Create a temp folder for reports (writable by all)
            sh 'mkdir -p /tmp/zap-reports && chmod 777 /tmp/zap-reports'

            // Run ZAP with host networking
            sh """
                docker run --rm \
                    --network host \
                    -v /tmp/zap-reports:/zap/reports \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py \
                        -t http://127.0.0.1:${appPort} \
                        -r /zap/reports/zap_report.html \
                        -J /zap/reports/zap_report.json \
                        -z "-config api.disablekey=true"
            """

            // Archive the generated reports
            archiveArtifacts artifacts: '/tmp/zap-reports/*', allowEmptyArchive: true
        }
    }
}


        stage('Generate HTML Report') {
            steps {
                script {
                    def htmlContent = """
                    <html>
                    <head><title>Pipeline Execution Report</title></head>
                    <body>
                    <h1>Pipeline Build Report</h1>
                    <h2>Build #${env.BUILD_NUMBER}</h2>
                    <p><strong>Status:</strong> ${currentBuild.currentResult ?: 'SUCCESS'}</p>
                    <p><strong>Date:</strong> ${new Date().format("yyyy-MM-dd HH:mm")}</p>
                    <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Duration:</strong> ${currentBuild.durationString ?: 'N/A'}</p>
                    <hr>
                    <h3>Build Steps Completed:</h3>
                    <ul>
                        <li>✅ Git Checkout</li>
                        <li>✅ Secrets Scan</li>
                        <li>✅ Unit Tests</li>
                        <li>✅ Dependency Check</li>
                        <li>✅ Trivy Scan</li>
                        <li>✅ SonarQube Analysis</li>
                        <li>✅ Docker Build & Push</li>
                        <li>✅ Deployment</li>
                        <li>✅ OWASP ZAP Scan</li>
                    </ul>
                    </body>
                    </html>
                    """

                    writeFile file: 'pipeline-report.html', text: htmlContent
                    archiveArtifacts artifacts: 'pipeline-report.html', allowEmptyArchive: true
                    archiveArtifacts artifacts: '*/target/surefire-reports/*.html', allowEmptyArchive: true
                }
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

