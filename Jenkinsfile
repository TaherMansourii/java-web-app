pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('e22f124e-2767-44cf-930d-23dd19c842a7')
        GITHUB_CREDENTIALS = credentials('da270b62-31a2-44ed-a570-c701a933abf6')
        DOCKER_IMAGE = '66raven99/java-web-app:latest'
        K8S_NAMESPACE = 'default'
        K8S_DEPLOYMENT = 'java-web-app'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }

    stages {
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



            post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs.'
        }
    }
}

